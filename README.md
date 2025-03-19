# NetBox

This Ansible role installs NetBox in an LXC Ubuntu 22.04 container with
PostgreSQL and Redis.

This role can setup a fresh NetBox instance or restore from an existing backup
(a backup taken with this role).


## Example playbook

On Ubuntu 22.04 in an LXC container.

```
- name: NetBox
  hosts: all

  tasks:
    # https://codeberg.org/ansible/python3
    - { import_role: { name: python3 },  become: true, tags: python3 }
    # https://codeberg.org/ansible/postgres
    - { import_role: { name: postgres }, become: true, tags: postgres }
    # https://codeberg.org/ansible/redis
    - { import_role: { name: redis },    become: true, tags: redis }
    # https://github.com/kommserv/netbox
    - { import_role: { name: netbox },   become: true, tags: netbox }
```

In `group_vars`/`host_vars` or similar:
```
psql_version: 14
psql_admin_username: postgres

redis:
  port: 6379
  packages:
    - redis

python_major_version: 3
python_version: "3.10"

netbox:
  version: "latest"
  config:
    default_allowed_hosts: [ netbox.example.se ]
    secret_key: "{{ lookup('community.general.passwordstore', 'netbox/secretkey') }}"
    database:
      name: netbox
      user: netbox
      pass: "{{ lookup('community.general.passwordstore', 'psql/users/netbox') }}"
      host: localhost
      port: 5432
      admin_username: "{{ psql_admin_username }}"
    superuser:
      user: "{{ lookup('community.general.passwordstore', 'netbox/users/admin subkey=usr') }}"
      email: "{{ lookup('community.general.passwordstore', 'netbox/users/admin subkey=email') }}"
      name: "{{ lookup('community.general.passwordstore', 'netbox/users/admin subkey=name') }}"
      pass: "{{ lookup('community.general.passwordstore', 'netbox/users/admin') }}"
      api_token: "{{ lookup('community.general.passwordstore', 'netbox/info subkey=SUPERUSER_API_TOKEN') }}"
    gunicorn:
      host: "0.0.0.0"
      port: 8001
    mail:
      server: "{{ lookup('community.general.passwordstore', 'netbox/info subkey=EMAIL_SERVER') }}"
      port: 587
      user: "{{ lookup('community.general.passwordstore', 'netbox/info subkey=EMAIL_USERNAME') }}"
      pass: "{{ lookup('community.general.passwordstore', 'netbox/info subkey=EMAIL_PASSWORD') }}"
      from: "{{ lookup('community.general.passwordstore', 'netbox/info subkey=EMAIL_FROM') }}"
    redis:
      port: 6379
      host: "localhost"
```

Apache virtualhost config on the reverse proxy (i.e., the LXC hypervisor):
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
   Alias /static /media/lxd/keelung/netbox/static/
   <Directory /media/lxd/keelung/netbox/static/>
      Options Indexes FollowSymLinks MultiViews
      AllowOverride None
      Require all granted
   </Directory>

   <Location /static>
      ProxyPass !
   </Location>

   RequestHeader set "X-Forwarded-Proto" expr=%{REQUEST_SCHEME}
   ProxyPass / http://<container-ip>:{{ netbox.gunicorn.port }}/
   ProxyPassReverse / http://<container-ip>:{{ netbox.gunicorn.port }}/

   SSLEngine on
   SSLCertificateKeyFile   /etc/letsencrypt/live/netbox.example.se/privkey.pem
   SSLCertificateFile      /etc/letsencrypt/live/netbox.example.se/cert.pem
   SSLCertificateChainFile /etc/letsencrypt/live/netbox.example.se/chain.pem

   ErrorLog ${APACHE_LOG_DIR}/netbox.example.se_error.log
   CustomLog ${APACHE_LOG_DIR}/netbox.example.se_access.log combined
</VirtualHost>
```

Note that the aliased paths only work because I **mounted** the `netbox/static`
path from *inside* the container onto the hypervisor, like this for example
(the permanent mount is done in fstab - see the `mounts` role config in luxor's `host_vars`):
```
$ sudo mkdir /media/lxd/<lxc-container-name>/netbox/static
$ bindfs --force-user=www-data --force-group=www-data \
  --create-for-user=www-data --create-for-group=www-data \
  /var/snap/lxd/common/mntns/var/snap/lxd/common/lxd/storage-pools/default/containers/<lxc-container-name>/rootfs/opt/netbox/netbox/static /media/lxd/<lxc-container-name>/netbox/static
```


## Links and notes

+ https://github.com/netbox-community/netbox
+ https://docs.netbox.dev/en/stable/getting-started/planning
