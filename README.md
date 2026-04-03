# System Setup

- Add `/etc/yum.repos.d/nginx.repo`

```
[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/10/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
priority=1
```

- Install NGINX

```bash
dnf install -y nginx
systemctl enable --now nginx
```

- Install EPEL

```bash
subscription-manager repos --enable codeready-builder-for-rhel-10-x86_64-rpms
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```

- Add SELinux policy

```bash
semanage fcontext -a -t httpd_log_t "/var/log/onlyoffice/documentserver/nginx.error.log"
```

- Install PostgreSQL
We are using the package from RHEL repositories here, because upstream PostgreSQL is not confined by SELinux.
Also, OnlyOffice doesn't even document which PostgreSQL version is supported, and their docker uses 15 as of this writing.

```bash
dnf install -y postgresql-server
/usr/bin/postgresql-setup --initdb
systemctl enable --now postgresql
```

Change authentication method for localhost from `ident` to `scram-sha-256` in `/var/lib/pgsql/data/pg_hba.conf`:

```
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

```bash
systemctl enable --now postgresql
```

- Create PostgreSQL user

```
[root@onlyoffice ~]# su - postgres
[postgres@onlyoffice ~]$ psql
postgres=# CREATE USER onlyoffice WITH PASSWORD 'REDACTED';
postgres=# CREATE DATABASE onlyoffice OWNER onlyoffice;
postgres=# exit
[postgres@onlyoffice ~]$ exit
[root@onlyoffice ~]#
```

- [Install RabbitMQ](https://www.rabbitmq.com/docs/install-rpm). Use the erlang version shipped by RabbitMQ - it is newer than what's available on Remi's repository.

```bash
sudo systemctl enable --now rabbitmq-server
```

- Install fonts

```bash
dnf install cabextract xorg-x11-font-utils https://sourceforge.net/projects/mscorefonts2/files/rpms/msttcore-fonts-installer-2.6-1.noarch.rpm -y
```

- Install OnlyOffice Document Server

```bash
dnf install https://download.onlyoffice.com/repo/centos/main/noarch/onlyoffice-repo.noarch.rpm -y
dnf install onlyoffice-documentserver -y
systemctl reload nginx
```

```
[root@onlyoffice conf.d]# documentserver-configure.sh
Configuring database access... 
Host: 127.0.0.1
Database name: onlyoffice
User: onlyoffice
Password: 

Trying to establish PostgreSQL connection... OK
Installing PostgreSQL database... OK
Configuring AMQP access... 
Host: 127.0.0.1
User: guest
Password: guest

Trying to establish AMQP connection... OK
Generating WOPI private key...Done
Generating WOPI public key...Done
Restarting services... OK
JWT is enabled by default. A random secret is generated automatically. Run the command '# documentserver-jwt-status.sh' to get information about JWT.
```

- Prevent new NGINX `default.conf` from being added

```bash
touch /etc/nginx/conf.d/default.conf
chattr +i /etc/nginx/conf.d/default.conf
```
