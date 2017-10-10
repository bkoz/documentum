# How I deployed OpenText Documentum on OpenShift 

## This README is not fully proof read - definitely a work in progress. It represents my notes from a project and is not a supported document from Red Hat, Inc. nor OpenText, Inc.

### Create the OpenShift project and grant privileges
```
PROJ=documentum
oc new-project $PROJ
oc adm policy add-scc-to-user anyuid -z default -n $PROJ
```
### Create the applications.

#### Push the images and create the image streams.

After exposing the OpenShift internal registry, use an external Docker client to load, tag 
and push the Documentum images to the OpenShift registry. This will create the necessary 
OpenShift image streams. Here is an example.

```REG_HOST=docker-registry-default.apps.fortnebula.com```

```
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
```

```
oc rsh $PG_POD_NAME mkdir /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chown -R postgres:postgres /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chmod 777 /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME ls -ld /var/lib/postgresql/data/db_centdb_dat.dat
```
#### Create the Content Server

```
PG_POD_IP=`oc get pods --selector=app=postgres --output=custom-columns=READY:.status.podIP --no-headers`
```
```
oc create -f cs.yaml
oc new-app documentum -p EXTERNALDB_IP=$PG_POD_IP -p EXTERNAL_IP=127.0.0.1
```

#### Create the DA Server

```oc create -f da.yaml```
```
DOCBROKER_IP=`oc get pods --selector=app=documentum --output=custom-columns=READY:.status.podIP --no-headers`
```

```oc new-app da -p DOCBROKER_IP={DOCBROKER_IP}```

### Notes

#### Lack of entropy work around

The content server installation scripts try to run ```rngd -b -r /dev/urandom -o /dev/random``` to increase the kernel entropy. As far as I
can tell, this requires full container privileges. I've tried the
following work arounds in OpenShift.

1) Pin all pods in the project to a given host by patching the 
deployment configuration then run ```rngd``` on that host. I've confirmed this works.

```
oc patch namespace documentum -p '{"metadata":{"annotations":{"openshift.io/node-selector": "kubernetes.io/hostname=ocpnode-1.fortnebula.com"}}}'
```

2) Add the ```privileged scc``` to the project. This also requires a patch to the deployment configuration. Until I work out the patch
command, the command and changes to the yaml are:

```oc oadm policy add-scc-to-user privileged -z default -n $PROJ```
```
spec:
      containers:
      - image: 172.30.23.146:5000/documentum/...
        imagePullPolicy: Always
        name: myapp
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

