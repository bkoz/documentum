# How I deployed OpenText Documentum on OpenShift 

## This README is a work in progress. It represents my notes from a project and is not a supported document from Red Hat, Inc. nor OpenText, Inc.

### Create the OpenShift project and grant it's default service account the```anyuid```scc.
```
PROJ=documentum
oc new-project $PROJ
oc adm policy add-scc-to-user anyuid -z default -n $PROJ
```

The content server installation scripts want to run```rngd -b -r /dev/urandom -o /dev/random```to increase the kernel entropy. As far as I
can tell, this requires running the pod in privileged mode (see notes below). The other option, which I've verified does work, is to pin all 
pods in the project to a given application node by patching the deployment configuration then running the above```rngd```command on that host.

```
oc patch namespace documentum -p '{"metadata":{"annotations":{"openshift.io/node-selector": "kubernetes.io/hostname=<your-app-node-name>"}}}'
```

### Create the applications.

#### Push the images and create the image streams.

After exposing the OpenShift internal registry, use an external Docker client to load, tag 
and push the Documentum images to the OpenShift registry. This will create the necessary 
OpenShift image streams. Here is an example.

```
REG_HOST=docker-registry-default.apps.fortnebula.com
docker load -i Contentserver_Centos.tar
docker login -u user -p token $REG_HOST
docker tag 93ca8e54e48e $REG_HOST/$PROJ/contentserver_centos:7.3.0000.0214
docker push $REG_HOST/$PROJ/contentserver_centos:7.3.0000.0214
```


Confirm the image streams were created.

```oc get is```
```
NAME                   DOCKER REPO                                       TAGS            UPDATED
contentserver_centos   172.30.23.146:5000/documentum/contentserver_centos   7.3.0000.0214   3 minutes ago
da_centos              172.30.23.146:5000/documentum/da_centos              7.3.0000.0074   44 seconds ago
```

#### Create the postgresql database

```oc new-app docker.io/postgres:9.6.1 -e POSTGRESQL_PASSWORD=password```

Wait for the postgres pod to become ready.

``` oc get pods```
```
NAME                 READY     STATUS    RESTARTS   AGE
postgres-1-2p143     1/1       Running   0          1m
```

Create a directory with the proper ownership and permissions.

```
PG_POD_NAME=`oc get pods --selector=app=postgres --output=custom-columns=NAME:.metadata.name --no-headers`
oc rsh $PG_POD_NAME mkdir /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chown -R postgres:postgres /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chmod 777 /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME ls -ld /var/lib/postgresql/data/db_centdb_dat.dat
```
#### Create the Content Server

```
PG_POD_IP=`oc get pods --selector=app=postgres --output=custom-columns=READY:.status.podIP --no-headers`
oc create -f cs.yaml
oc new-app documentum -p EXTERNALDB_IP=$PG_POD_IP -p EXTERNAL_IP=<your-app-nodes-ip-address>
```

#### Create the DA Server

```
oc create -f da.yaml
DOCBROKER_IP=`oc get pods --selector=app=documentum --output=custom-columns=READY:.status.podIP --no-headers`
oc new-app da -p DOCBROKER_IP={DOCBROKER_IP}
```

### Notes

#### Running with a privileged SCC.

To run with the```privileged scc```, add the scc to the project's default service account then patch the deployment configuration as follows. 


```oc oadm policy add-scc-to-user privileged -z default -n $PROJ```

Until I work out the patch command, the changes to the deploymenmt configuration yaml code snippet is:

```
spec:
      containers:
      - image: ...
        ...
        ...
        ...
        ...
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

