  # Collabora server on CTS

  to deploy a collabora server image in the openshift cluster in th CTS, you need to rebuild the collabora image with the UID that suites the UID range of the project.

  [ read about this in depth hear:
  https://wiki.linnovate.net/en/marketpalce/nextcloud-on-CTS]]

  this is the dockerfile you need to use to build the image
  > note: it could be that some of the commands are not needed, but this docker file works fine!

  ```bash
  FROM <base collabora image>

  # you need to rus as user root to add users and groups
  USER root

  # add a new user with a UID that suets the UID that runs inside the open
  RUN groupadd --gid 1000710000 newcool
  RUN useradd -l -u 1000710000 -g 1000710000 -d /opt/cool -s /usr/sbin/nologin newcool
  #=============

  RUN chown -R newcool:newcool /opt/cool/\
      && chown -R newcool:newcool /etc/coolwsd/ \
      && chown -R newcool:newcool /usr/share/coolwsd/

  RUN chown -R newcool:newcool /opt/cool \
      && chown -R newcool:newcool /etc/coolwsd \
      && chown -R newcool:newcool /usr/share/coolwsd

  RUN chown -R newcool:newcool /usr/bin/coolmount
  RUN find / -user 104 -exec chown -h 1002890000 {} \; || true

  RUN userdel cool
  RUN usermod -l cool newcool
  RUN groupmod -n cool newcool

  RUN chown -R cool:cool /usr/bin/
  RUN chown cool:cool /usr/bin/coolmount
  RUN chown -R cool:cool /opt/cool/child-roots/
  RUN chown cool:cool /usr/bin/coolforkit

  RUN setcap cap_sys_chroot+ep /usr/bin/coolforkit
  RUN setcap cap_mknod+ep /usr/bin/coolforkit
  RUN setcap cap_fowner+ep /usr/bin/coolforkit
  RUN setcap cap_chown+ep /usr/bin/coolforkit

  USER cool




