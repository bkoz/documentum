# Installing Documentum on OpenShift

## This is not a supported procedure from Red Hat, Inc. nor OpenText, Inc.

The README and template files represent my notes to deploy the Documentum content and 
admin servers on OpenShift. 

### Create the OpenShift project 

Grant the project's default service account the `anyuid` scc. All `oc adm` commands need to be run by the OpenShift cluster administrator.

```
PROJ=documentum
oc new-project $PROJ
oc adm policy add-scc-to-user anyuid -z default -n $PROJ
```

The content server installation scripts want to run `rngd -b -r /dev/urandom -o /dev/random` to increase the kernel entropy. 
As far as I can tell, this requires running the pod in privileged mode (see notes at the end). The following approach, which I've verified does work, is to pin all pods in the project to a given application node by patching the deployment configuration's `node-selector` with the hostname then run the above `rngd` command on that host. This approach also satisfies the requirement that the container host  `EXTERNAL_IP` be passed into the content server pod before the installer scripts are run.

Example patch command (set APP-NODE-HOSTNAME to match your environment):
```
oc patch namespace documentum -p '{"metadata":{"annotations":{"openshift.io/node-selector": "kubernetes.io/hostname=APP-NODE-HOSTNAME"}}}'
```

### Create the OpenShift objects.

#### Push the container images and create the image streams.

Once a user has been [configured to access the OpenShift internal registry](https://docs.openshift.com/container-platform/3.6/install_config/registry/accessing_registry.html#access-user-prerequisites), 
use an external Docker client to push the Documentum images to the OpenShift registry. This will create 
the necessary OpenShift image streams. Below is an example (set `REG_HOST` to match your environment).

```
TOKEN=`oc whoami -t`
REG_HOST=docker-registry-default.apps.mydomain.com
docker load -i Contentserver_Centos.tar
docker login -u user -p $TOKEN $REG_HOST
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

Set `EXTERNAL_IP` to the IP of your app node.

```
EXTERNALDB_IP=`oc get pods --selector=app=postgres --output=custom-columns=READY:.status.podIP --no-headers`
oc create -f cs.yaml
oc new-app documentum -p EXTERNALDB_IP=$EXTERNALDB_IP -p EXTERNAL_IP=<your-app-nodes-ip-address>
```
If you need to examine logs or debug the installation, use `oc rsh`  to connect to the pod's
namespace.

```
CS_POD_NAME=`oc get pods --selector=app=documentum --output=custom-columns=NAME:.metadata.name --no-headers`
oc rsh $CS_POD_NAME bash
```

#### Create the DA Server

```
oc create -f da.yaml
DOCBROKER_IP=`oc get pods --selector=app=documentum --output=custom-columns=READY:.status.podIP --no-headers`
oc new-app da -p DOCBROKER_IP=$DOCBROKER_IP
```

### Notes

#### Platform specifics
This test was performed using OpenShift v3.6 running on RHEL Server 7.4. Using RHEL Atomic host was not successful. The content server installation scripts produced errors that 
are being investigated.

#### Running with a privileged SCC.

To run with the `privileged scc`, add the scc to the project's default service account then patch the deployment configuration as follows. 


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

