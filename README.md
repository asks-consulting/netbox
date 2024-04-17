# NetBox

This Ansible role is designed to run
[NetBox](https://github.com/netbox-community/netbox)
in an LXC container running Ubuntu 20.04 with
[Python 3.8](https://codeberg.org/ansible/python3),
[PostgreSQL 12.10](https://codeberg.org/ansible/postgres)
and [Redis 5.0](https://codeberg.org/ansible/redis).

Since the LXC server can see the filesystem of the container, we can
save on some resources by not running any web server in the container itself.
This approach was totally inspired by [@candlerb's setup](https://github.com/netbox-community/netbox/discussions/8598#discussioncomment-2147712).

This role handles all steps of setting up a fresh NetBox instance or restoring
from an existing database dump non-interactively, including superuser creation.

This role employs an ugly hack in `settings.py` to force NetBox to
use Redis via socket instead of port. I happen to have other services
on the same container that already used Redis socket, so that's why.
I don't expect this hack to survive a NetBox upgrade, so beware!
Hopefully NetBox will support Redis sockets soon and we can retire this hack.

I am grateful to the stellar documentation and helpful support forums
of the NetBox project, without which I could not have completed this role
nor kept NetBox running for several years.


## Apache virtualhost config on LXC server

```
<VirtualHost *:80>
   ServerAdmin webmaster@example.se
   ServerName netbox.example.se

   Redirect permanent / https://netbox.example.se/

   Loglevel info
   ErrorLog ${APACHE_LOG_DIR}/netbox.example.se_error.log
   CustomLog ${APACHE_LOG_DIR}/netbox.example.se_access.log combined
</VirtualHost>

<VirtualHost *:443>
   ServerAdmin webmaster@example.se
   ServerName netbox.example.se

   ProxyPreserveHost On
   Alias /static /var/lib/lxd/containers/<container-name>/rootfs/opt/netbox/netbox/static/
   <Directory /var/lib/lxd/containers/<container-name>/rootfs/opt/netbox/netbox/static/>
      Options Indexes FollowSymLinks MultiViews
      AllowOverride None
      Require all granted
   </Directory>

   <Location /static>
      ProxyPass !
   </Location>

   RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
   ProxyPass / http://<container-ip>:8001/
   ProxyPassReverse / http://<container-ip>:8001/

   SSLEngine on
   SSLCertificateKeyFile   /etc/letsencrypt/live/netbox.example.se/privkey.pem
   SSLCertificateFile      /etc/letsencrypt/live/netbox.example.se/cert.pem
   SSLCertificateChainFile /etc/letsencrypt/live/netbox.example.se/chain.pem

   ErrorLog ${APACHE_LOG_DIR}/netbox.example.se_error.log
   CustomLog ${APACHE_LOG_DIR}/netbox.example.se_access.log combined
</VirtualHost>
```


## Example playbook

```
- name: Container NetBox
  hosts: netbox

  tasks:
    # https://codeberg.org/ansible/common
    - name: "setup: common"
      import_role:
        name: common
      become: true
      tags: common
   # https://codeberg.org/ansible/common-systools
    - name: "setup: common-systools"
      import_role:
        name: common-systools
      become: true
   # https://codeberg.org/ansible/locales
    - name: "setup: locales"
      import_role:
        name: locales
      become: true
      tags: locales
    # https://codeberg.org/ansible/ssh
    - name: "ssh"
      import_role:
        name: ssh
      become: true
   # https://codeberg.org/ansible/dotfiles
    - name: "dotfiles"
      import_role:
        name: dotfiles
      tags: dotfiles
   # https://github.com/kommserv/netbox
    - name: "NetBox"
      import_role:
        name: netbox
      become: true
      tags: netbox
```

Playbook-level variables:
```
psql_version: 12
redis_socket.path: "/var/run/redis/redis-server.sock"
php_version: 7.3
backup_root_path: /storage/backup/
```
