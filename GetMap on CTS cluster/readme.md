
# How to deploy GetMap on CTS cluster

When deploying applications on the CTS Openshift cluster, there is a significant challenge that we don't encounter in the "outside" environment: the inability to run containers as the root user. This limitation presents two main problems:

1.  Binding port 80 to your application becomes impossible, as only root users have the necessary permissions to do so. Consequently, if your application is designed to expose port 80, the deployment will fail with an error message stating "permission denied" when attempting to bind the port.
    
2.  Another common issue arises when the Openshift user running the container tries to access files for reading, writing, or executing that are not owned by the user or their group. In such cases, the user will encounter a "permission denied" error.
    
Now, let's move on to the solution for these challenges.

> ### importent! you can skip this whole article, and just rebuild the images whith the Dockerfiles - it will work!

> I wrote here a small introduction on this topic in openshift. you can skip this if you want, and go ahead to the solution part.

## Introduction:
OpenShift is designed to run containers with a unique User ID (UID) and Group ID (GID) from a pre-defined range of IDs in every project within the cluster. Even if you attempt to run a container as the "root" user, by adding a security context with `runAsUser: 0` - it won't work.

To overcome this issue, an OpenShift administrator can create a service account with a Security Context Constraint (SCC) that permits running containers as `anyuid`. This SCC grants a specific level of access and control over the container and its environment.

Here's an example of how to create a service account with an SCC that allows running containers as "anyuid":

```bash
oc create sa anyuid-sa # replace "anyuid-sa"  with any name you want
oc adm policy add-scc-to-user anyuid -z anyuid-sa # replace "anyuid-sa" with the name of the sa form the last command
```
The above commands create a service account named "anyuid-sa" and associate it with the "anyuid" SCC.

You can now use this service account in your deployment YAML file to run the container with the necessary permissions:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: anyuid-sa
      containers:
      - name: my-container
        image: my-image
        securityContext:
          runAsUser: 0
```
to read more about service account's & Security Context Constraint:
https://developer.ibm.com/learningpaths/secure-context-constraints-openshift/
*** ****
**the problem in CTS:**
in the CTS cluster we don't have an admin Permission.
due to that fact - we can't use a service account diffrent then the default.
any container you will run in the cluster, will run as the UID that's asingnd to it by openshift, with no way to change it.
becuse of that, the container will allwase run into permitions problems, becuse the container trys to read / write / execute files that he dosen't own.

**End of Introduction!** 
*** ****
# Solution
GetMap uses 3 containers:
1. getmap-service
2. SEQ
3. postgis

evrey container hase a diffrent solution for the 2  problemes.

## Problem #1: binding port 80 
there is no way to expose port 80 inside the container (the service and the route **can** expose port 80).
so to run around this problem, you need to expose it on a different port.
### Getmap service:
GetMap is a `.net` app, and as that you can use an environment variable to expose the app on a different port.
Add this variable to your deployment (in the "environment" section in the deployment, or in the config map that is attached to the deployment:
```bash
ASPNETCORE_URLS = http://0.0.0.0:8080
```
this will expose the app on port 8080.
you can now create a service that uses port 8080 as a "target port" and expose it on port 80:
```bash
apiVersion: v1
kind: Service
metadata:
	name: getmapservice-svc
spec:
	ports:
		- name: http
		port: 80
		protocol: TCP
		targetPort: 8080
	selector:
		app: getmapservice
```
### SEQ:
the SEQ container exposes its UI in port 80. to change that you need to add a environment variable:
```
SEQ_API_LISTENURIS = 8080
```
> Now you can expose the container on port 80 with a service - like in getmap 

### Postgis:
Postgis runs as default on port 5432, so there is no need to take action.


> ### You can also add the environment variables inside the image itself (via the `Dockerfile` during the build, see next step.
## Problem #2: "permission denied" error
To avoid getting the error "permission denied", you need to re-build the image, and change the ownership of the files to the `UID` of the open shift user.

#### stage 1: get the UID & GID of the project.
you need to create a new user in the container, that has the same UID as the one the cluster uses, and then change ownership of the app files to the new user you just created.

first, find the UID & GID user of the project:
```bash
oc get project <project name> -o yaml | grep "openshift.io/sa.scc.uid-range" | cut -d: -f2 | cut -d/ -f1 | sed -e 's/^[[:space:]]*//'

output: 100071000
```
another way to see that info, is in the UI of openshift:
on the left menu go to `home > projects`, find your project, and look at the YAML file:
```
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: getapp
  uid: 7cadc868-a2c9-4179-90e0-5d99ba6bc581
  resourceVersion: '11340160'
  creationTimestamp: '2023-02-12T13:52:44Z'
  labels:
    kubernetes.io/metadata.name: getapp
    olm.operatorgroup.uid/670b038d-8b91-4d24-8fb7-6999b6d9224d: ''
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: v1.24
  annotations:
    openshift.io/description: ''
    openshift.io/display-name: ''
    openshift.io/requester: 'kube:admin'
    openshift.io/sa.scc.mcs: 's0:c27,c4'
    openshift.io/sa.scc.supplemental-groups: 1000710000/10000
    openshift.io/sa.scc.uid-range: 1000710000/10000
```
in the last line of the code you can see the `UID` that is `1000710000`
after you have the UID of the project, you can continue to the next step: re-build the image.
#### stage 2: re-build the image
### GetMap
use this docker file to rebuild the image:
```
FROM docker harbor.getapp.sh/getapp/getmapservice:latest
USER root
#=============
RUN groupadd --gid 10000710000 getmap
RUN useradd -l -u 10000710000 -g 10000710000 -d /var/www -s /usr/sbin/nologin getmap

#=============
# optional - you can pass it also into the deployment itself
VAR ASPNETCORE_URLS=http://0.0.0.0:8080
RUN chown -R getmap:getmap /app/

```
> Replace the number `1000710000` with the UID of the project (see stage #1)

### SEQ
use this docker file to rebuild the image:
```
FROM datalust/seq:latest
USER root
#=============
RUN groupadd --gid 10000710000 seq
RUN useradd -l -u 10000710000 -g 10000710000 -d /var/www -s /usr/sbin/nologin seq
#=============
# optional - you can pass it also into the deploymant itself
VAR SEQ_API_LISTENURIS=8080
RUN chown -R seq:seq /seqsvr/
```
> Replace the number `1000710000` with the UID of the project (see stage #1)

### Postgis
use this docker file to rebuild the image:
```
FROM postgis/postgis:15-3.3
USER root
#=============
RUN groupadd --gid 10000710000 postgis
RUN useradd -l -u 10000710000 -g 10000710000 -d /var/www -s /usr/sbin/nologin postgis
#=============
RUN mkdir /var/lib/postgresql-data/
RUN chown -R postgis:postgis /var/lib/postgresql-data/
```
> Replace the number `1000710000` with the UID of the project (see stage #1)


> After building the images "outside", you need to save them as a tar file with `docker save` and then put them on a disk on key, and go to the Halbana computer.

**Importent update:** using the postgis image as a data base is not best practice, not even in a none prod environment.
The correct approach to connect to a PostGIS database is using the "portal" data base:
in the CTS go to "portal" and open a postgers database. you will need to open a ticket and ask to add the GIS extention to this data base. 
