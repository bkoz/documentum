# How I deployed OpenText Documentum on OpenShift
## This represents my notes from a project but is not a supported document from Red Hat, Inc.
### Creating the OpenShift project
```oc new-project documentum```

### Creating the postgresql database

```oc new-app docker.io/postgres:9.6.1 -e POSTGRESQL_PASSWORD=password```

Wait for the postgres pod to become ready (READY = 1/1)
``` oc get pods```

Once the postgresql pod is ready, a directory with the proper permissions must be created.

```
export PG_POD_NAME=`oc get pods --selector=app=postgres --output=custom-columns=NAME:.metadata.name --no-headers`
oc rsh $PG_POD_NAME mkdir /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chown -R postgres:postgres /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME chmod 777 /var/lib/postgresql/data/db_centdb_dat.dat
oc rsh $PG_POD_NAME ls -ld /var/lib/postgresql/data/db_centdb_dat.dat
```


### Creating the Content Server


### Creating the DA Server
