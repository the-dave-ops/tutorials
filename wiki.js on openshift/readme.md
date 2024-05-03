# Deploying wiki.js on CTS
When deploying applications on the CTS Openshift cluster, there is a significant challenge that we don't encounter in the "outside" environment: the inability to run containers as the root user. This limitation presents two main problems:

1.  Binding port 80 to your application becomes impossible, as only root users have the necessary permissions to do so. Consequently, if your application is designed to expose port 80, the deployment will fail with an error message stating "permission denied" when attempting to bind the port.
    
2.  Another common issue arises when the Openshift user running the container tries to access files for reading, writing, or executing that are not owned by the user or their group. In such cases, the user will encounter a "permission denied" error.

3. a similar issue is when trying to create a new file, with `touch` or `>` and when trying to create a new directory with `mkdir`.
    
Now, let's move on to the solution to these challenges.

for additional info - go to: 
https://wiki.linnovate.net/en/clients/Getmap/InstallGetmapOnCTS

## Step 1: download the language files
Follow the steps in this article:
https://docs.requarks.io/install/sideload

for now, you don't need to accomplish all of the steps in the article. for now just download the language files and the main configuration file. 
for example, for a Hebrew and English installation, you will need to have these 3 files in the directory:
`he.json`
`en.json`
`locales.json`

> Later on in the docker build we will copy these files into the image, to the exact directory. no need to take any action for now.
{.is-info}



## Step 2: get the UID of the current open shift project
you will need to create a new user in the image, that has the same UID as the one the cluster uses.

to find the UID & GID user of the project:

```bash
oc get project <project name> -o yaml | grep "openshift.io/sa.scc.uid-range" | cut -d: -f2 | cut -d/ -f1 | sed -e 's/^[[:space:]]*//'

output: 100071000
```
another way to see that info is in the UI of open shift:
on the left menu go to `home > projects`, find yor project, and look at the YAML file (see lines 20 21):

![screenshot_from_2023-03-29_13-49-52.png](/screenshot_from_2023-03-29_13-49-52.png)

## Step 3: create a user-change.sh file

open shift uses a high UID range. all the UID's are 10-figure numbers.
in alpine linux you cant create a UID higher than 256000. the wiki.js image is based on Alpine, so by default, you can't create user's like 1000710000.
we can overcome this restriction by editing the /etc/passwd file manually.
we will use a script in the docker build procces to do this.

**copy this code, and save it in the current directory.**

```
#!/bin/bash

# UID & GID
uid=1002990000
# group name & user name
username=wikijs

# Add the user to /etc/passwd
echo "${username}:x:${uid}:${uid}:${username}:/home/${username}:/sbin/nologin" >> /etc/passwd

# Add the user to /etc/group
echo "${username}:x:${uid}:" >> /etc/group
```
change the value of the uid (100290000) to the UID you got in step 2.

## step 4: create a config.yml file

to get the language files to work on an offline installation, we need to use a custom `config.yml` file.

copy this code and save it as `config.yml`
```
port: 3000
bindIP: 0.0.0.0
db:
  type: $(DB_TYPE)
  host: '$(DB_HOST)'
  port: $(DB_PORT)
  user: '$(DB_USER)'
  pass: '$(DB_PASS)'
  db: $(DB_NAME)
  storage: $(DB_FILEPATH)
  ssl: $(DB_SSL)
ssl:
  enabled: $(SSL_ACTIVE)
  port: 3443
  provider: letsencrypt
  domain: $(LETSENCRYPT_DOMAIN)
  subscriberEmail: $(LETSENCRYPT_EMAIL)
logLevel: $(LOG_LEVEL:info)
logFormat: $(LOG_FORMAT:default)
ha: $(HA_ACTIVE)
language:
  default: en
  supported: [en, he]
offline: $(OFFLINE_ACTIVE)
```
## step 5: create a Dockerfile

this is the last step: buiding the new image with the files we created in the previous step.  

first, copy this code and save it as `Dockerfile`
change the UID in line 11 (100290000) to the uid from step 2.
```
FROM requarks/wiki:2

USER root

# using the script to create a new user and group
COPY user-change.sh /user-change.sh
RUN chmod +x /user-change.sh
RUN /bin/sh /user-change.sh

# copy lang files
RUN mkdir /wiki/data/sideload
COPY *.json  /wiki/data/sideload/

# cahge permitions of the app files
RUN chgrp -R 1002990000 /wiki /logs && \
    chmod -R g=u /wiki /logs

# just for documentation
COPY Dockerfile /Dockerfile

# the configuration file
COPY config.yml /wiki/config.yml

USER 1002990000
```

## step 6: build the image
by now you will have these files in the same directory:
`Dockerfile`
`he.json`
`en.json`
`locales.json`
`config.yml`
`user-change.sh`

if you have all of this files - you are redey for the build stage:
```
docker build -t wikijs-cts:1 .
```

## Step 7: deployment configuration

when deploying wiki.js you need to connect to a Postgres DB.
after opening the DB, we need to add the details as environment variables to the wiki.js deployment

these of the env you need to use:
(change the values to your DB details)
```
DB_TYPE: postgres
DB_HOST: db
DB_PORT: 5432
DB_USER: wikijs
DB_PASS: wikijsrocks
DB_NAME: wiki 
```