# Best way to create a new postgres cluster with crunchy data operator
when creating a new postgres cluster with the UI in openshift, is wont work out of the box. the best way is to apply this yaml, and that will do the job. in this way you can include the file in a chart, and create an new cluster for a new projct in a easy way.
```
kind: PostgresCluster
apiVersion: postgres-operator.crunchydata.com/v1beta1
metadata:
name: "postgres-{{ .Values.configmap }}"
spec:
backups:
pgbackrest:
repos:
- name: repo1
volume:
volumeClaimSpec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 1Gi
instances:
- dataVolumeClaimSpec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 1Gi
replicas: 1
databases:
- name: {{ .Values.configmap }}
template: template1
owner: postgres
lcCollate: "en_US.UTF-8"
lcCtype: "en_US.UTF-8"
patroni:
dynamicConfiguration:
postgresql:
pg_hba:
- host all all 0.0.0.0/0 trust
- host all postgres 127.0.0.1/32 md5
postgresVersion: 15
```
this will also create a new database with the name in the "name" filed in line 26.
the database will have a prefix of "postgres" + the name in the filed, so for example if "{{ .Values.configmap }}" = getapp, the full name of the database will be `postgres-getapp`.
the lines 31-36 create a pg_hba.conf file that allows connections to the database.