# How to insatll nextcloud on openshift cluster
### importent note: this artical refres to nextcloud, but is true for other apps as well.



**Introduction:**
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
**the problem in a restricte:**
in a restricte cluster we don't have an admin Permission.
due to that fact - we can't use a service account diffrent then the default.
any container you will run in the cluster, will run as the UID that's asingnd to it by openshift, with no way to change it.
becuse of that, the container will allwase run into permitions problems, becuse the container trys to read / write / execute files that he dosen't own.
*** ****
**the solution:**
###### stage 1: get the UID & GID of the project.
you need to create a new user in the container, that has the same UID as the one the cluser uses, and then change ownership of the app files to the new user you just created.

first, find the UID & GID user of the project:

```bash
oc get project <project name> -o yaml | grep "openshift.io/sa.scc.uid-range" | cut -d: -f2 | cut -d/ -f1 | sed -e 's/^[[:space:]]*//'

output: 100071000
```
another way to see that info, is in the UI of openshift:
on the left menu go to `home > projects`, find yor project, and look at the YAML file (see lines 20 21):

![screenshot_from_2023-03-29_13-49-52.png](/screenshot_from_2023-03-29_13-49-52.png)

###### stage 2: modify the image.
then, rebuild the image with adding a new UID, and change ownership of the files: 

*DockerFile:*
```bash
# this is a docker file that allows you to deploy nextcloud
# in a open shift cluster that has a SCC that does not allow you to\
# run the container as user mode.
# pre-runinng:
#   you need to find the UID that is assigned to containers in your open shift project.

FROM nextcloud:25

# you need to rus as user root to add users and groups
USER root

# add a new user with a UID that suets the UID that runs inside the open
RUN groupadd --gid 1000710000 nextcloud
RUN useradd -l -u 1000710000 -g 1000710000 -d /var/www -s /usr/sbin/nologin nextcloud
#=============

RUN chown -R nextcloud:nextcloud /var/www/
RUN chown -R nextcloud:nextcloud /usr/local/etc/php/conf.d/

# install plugins:
#RUN sudo -u nextcloud php occ app:install user_social_login
#RUN sudo -u nextcloud php occ app:install richdocumentscode

# configure nextcloud to run with a different port number
RUN sed -i 's/Listen 80/Listen 8080/g' /etc/apache2/ports.conf
RUN sed -i 's/:80/:8080/g' /etc/apache2/sites-enabled/000-default.conf
USER nextcloud
ENTRYPOINT [ "/bin/bash", "/entrypoint.sh" ]
CMD [ "apache2-foreground" ]
```
### very importent: in the `useradd` commad, you must use the `-l` flag! the build will fail if not!
you can read about this issue here:
https://github.com/moby/moby/issues/5419
https://stackoverflow.com/questions/48671214/docker-image-size-for-different-user
###### stage 3: deplyment with the new image
upload the image to yor image reopsitory, and use it in yor deployment.

## Alpine based images
When using an image that is based on Alpine, this solution will run into a problem because Alpine does not allow to create users with a UID higher than 256000. You can overcome this restriction by editing the /etc/passwd file manually. For the exact way to do this, go to here: 
https://stackoverflow.com/questions/41807026/cant-add-a-user-with-a-high-uid-in-docker-alpine#:~:text=Here%20is%20a%20working%20but%20dirty%20workaround%2C%20by%20manually%20creating%20the%20user%2C%20using%20%24UID_TO_SET%20as%20the%20bash%20variable%20containing%20the%20high%20UID%20to%20set%3A

based on that info you can use a bash script in the `docker build` process to to this:
```bash
#!/bin/bash

# UID & GID
uid=1000710000
# group name & user name
username=nextcloud

# Add the user to /etc/passwd
echo "${username}:x:${uid}:${uid}:/home/${username}:/sbin/nologin" >> /etc/passwd

# Add the user to /etc/group
echo "${username}:x:${uid}:" >> /etc/group

```

> after  a while i found this article that addreses some of the issues that i described here, and give the same solution:
https://birthday.play-with-docker.com/run-as-user/
{.is-info}