# netbox

It is time to migrate my outdated Netbox instance running in Docker on `armant`
to a more robust configuration managed by Ansible on an LXC container.


## Status of netbox instance on armant

Running Netbox v2.5.12 (7233187d939c). Amazingly, it is still running
without issues.

Stack: Ubuntu 18.04.5, Docker v20.10.5, docker-compose v1.24.0
Inside container: PostgreSQL v10, nginx, Python, Redis, gunicorn

```
root@armant:~# docker ps 
CONTAINER ID  IMAGE                         COMMAND                 CREATED       STATUS        PORTS                           NAMES
e3a0baac392d  nginx:1.15-alpine             "nginx -c /etc/netbo…"  2 years ago   Up 11 months  80/tcp, 0.0.0.0:8484->8080/tcp  netbox-docker_nginx_1
7233187d939c  netboxcommunity/netbox:latest "/opt/netbox/docker-…"  2 years ago   Up 11 months                                  netbox-docker_netbox_1
ade44d4114e2  netboxcommunity/netbox:latest "python3 /opt/netbox…"  2 years ago   Up 11 months                                  netbox-docker_netbox-worker_1
92ec3eb028ff  postgres:10.4-alpine          "docker-entrypoint.s…"  2 years ago   Up 11 months  5432/tcp                        netbox-docker_postgres_1
51a82cdbe60b  redis:4-alpine                "docker-entrypoint.s…"  2 years ago   Up 11 months  6379/tcp                        netbox-docker_redis_1
root@armant:~# docker ps --format "{{.Names}}"
netbox-docker_nginx_1
netbox-docker_netbox_1
netbox-docker_netbox-worker_1
netbox-docker_postgres_1
netbox-docker_redis_1
root@armant:~# ll /var/lib/docker/volumes/
total 76
drwx-----x 11 root root  4096 Mar 11  2021 ./
drwx--x--x 14 root root  4096 Mar 11  2021 ../
brw-------  1 root root  8, 2 Mar 11  2021 backingFsBlockDev
-rw-------  1 root root 65536 Mar 11  2021 metadata.db
drwxr-xr-x  3 root root  4096 May  7  2019 monica_data/
drwxr-xr-x  3 root root  4096 May  7  2019 monica_mysql/
drwxr-xr-x  3 root root  4096 May  7  2019 monica_public/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-media-files/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-nginx-config/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-postgres-data/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-redis-data/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-report-files/
drwxr-xr-x  3 root root  4096 May  6  2019 netbox-docker_netbox-static-files/
```
https://stackoverflow.com/questions/31887258/get-docker-container-names

Dump the pgsql database:
```
docker exec -i netbox-docker_postgres_1 /bin/bash -c
"PGPASSWORD=<pg_password> pg_dump --username <pg_username> <database_name>" > 
/desired/path/on/your/machine/dump.sql
```
OK, this worked!

We should also copy our media directory.
https://netbox.readthedocs.io/en/stable/installation/upgrading/
These seem to be saved in `/var/lib/docker/volumes/netbox-docker_netbox-media-files/_data/image-attachments/`,
which contains two photos. Will migrate these manually. OK.







## Setup Netbox v2.5.12 on container with PostgreSQL v10.4

The idea is the following. Setup Netbox v2.5.12 on a stack as similar as possible 
to before, but in a LXC container.

I don't think this container will need neither Apache nor PHP. Let's test without them.
https://netbox.readthedocs.io/en/stable/installation/5-http-server/

OK, installed `psql v10.20`, `redis-server v5.0.7`, `python 3.7` on 
LXC container Ubuntu 20.04.4.

WARNING! Netbox v2.5.12 requires psycopg2-binary v2.7.6.1, which 
**is known to be incompatible** with Python 3.8.
We must stick with Python v3.7 or earlier (`armant` was on 3.6.9)!
https://github.com/psycopg/psycopg2/issues/854#issuecomment-611791946

**We will need to use deadsnakes and specify v3.7**
Setting it in our subgroup should override the same setting the parent group `group_vars/lxd_containers`?

In either the `pip install requirements.txt` step or the `upgrade.sh` step
we install the pip package markupsafe, which since it's version is not specified
in the requirements file gets us markupsafe 2.1.0, which apparently contains
a known breaking change with earlier versions.

This caused `upgrade.sh` to fail with the error:
```
ImportError: cannot import name 'soft_unicode' from 'markupsafe' 
```
Let's see if pinning `markupsafe<2.1.0` in `requirements.txt` fixes it.

https://github.com/aws/graph-notebook/issues/267
https://github.com/pallets/markupsafe/issues/282

+ Install Netbox v2.5.12.
https://github.com/netbox-community/netbox/archive/refs/tags/v2.5.12.tar.gz

+ Test, OK works!

+ Consider gunicorn and Apache reverse proxy setup...

On recent Netbox versions it seems gunicorn is installed in Netbox's python venv.
But this does not seem to be the case for v2.5.12. On my old instance, gunicorn 
is found at `/usr/bin/gunicorn` (inside the Docker container).

OK, installed gunicorn, configured systemd services.
Managed to run Netbox behind reverse proxy on `luxor` without needing another 
local proxy on the container itself  (vhost on `luxor`):
```
<VirtualHost *:80>
	ServerAdmin {{ SUPERUSER_EMAIL }}
	ServerName {{ ALLOWED_HOSTS[0] }}

	Redirect permanent / https://{{ ALLOWED_HOSTS[0] }}/

	ErrorLog ${APACHE_LOG_DIR}/{{ ALLOWED_HOSTS[0] }}_error.log
	CustomLog ${APACHE_LOG_DIR}/{{ ALLOWED_HOSTS[0] }}_access.log combined
</VirtualHost>

<VirtualHost *:443>
	ServerAdmin {{ SUPERUSER_EMAIL }}
	ServerName {{ ALLOWED_HOSTS[0] }}

   ProxyPreserveHost On
   Alias /static /var/lib/lxd/containers/netbox/rootfs/opt/netbox/netbox/static/
   <Directory /var/lib/lxd/containers/netbox/rootfs/opt/netbox/netbox/static/>
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
	SSLCertificateKeyFile   /etc/letsencrypt/live/{{ ALLOWED_HOSTS[0] }}/privkey.pem
	SSLCertificateFile      /etc/letsencrypt/live/{{ ALLOWED_HOSTS[0] }}/cert.pem
	SSLCertificateChainFile /etc/letsencrypt/live/{{ ALLOWED_HOSTS[0] }}/chain.pem
	
	ErrorLog ${APACHE_LOG_DIR}/{{ ALLOWED_HOSTS[0] }}_error.log
	CustomLog ${APACHE_LOG_DIR}/{{ ALLOWED_HOSTS[0] }}_access.log combined
</VirtualHost>
```
Note the neat trick of pointing the static files directly out from the the container's 
filesystem. Thanks to @candlerb for that neat solution! 

> The proxy serves static files, and anything that's not under `/static/
> it proxies to gunicorn on port `8001`.
> https://github.com/netbox-community/netbox/discussions/8598

+ Import the database. OK!

+ Test Netbox. Was migration successful? Looks good.

+ Now try again with PostgreSQL v12, Python 3.8 and the latest Netbox version 
  (restore from dump as before).
  This had to be done in steps:

  + imported database dump from v2.5.12 into Netbox v2.11.12 - OK.
  + created new database dump: `sudo su - postgres && pg_dump netbox > netbox-v2.11.12.sqlpg_dump netbox > netbox-v2.11.12.sql`. Saved.
  


Now we should be able to safely migrate Netbox to its final home on `taipei`,
targeting the latest version of Netbox.
**Before touching the container, snapshot it!**





## Refs

+ https://davejansen.com/how-to-dump-and-restore-a-postgresql-database-from-a-docker-container/
