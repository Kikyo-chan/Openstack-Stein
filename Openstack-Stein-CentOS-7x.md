## Openstack Stein on Centos 7x

### 1. Mô hình

<img src="/img/1.jpg">

## 2. Cài đặt môi trường

- Chạy trên tất cả các node

```
echo "192.168.239.180 controller" >> /etc/hosts
echo "192.168.239.181 compute1" >> /etc/hosts
```

- Trên controller config network & set hostname

`hostnamectl set-hostname controller`

```
[root@controller ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="b5a22fbe-c1b7-4676-916a-50ff4513947b"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.239.180"
PREFIX="24"
GATEWAY="192.168.239.1"
DNS1="8.8.8.8"
```

```
[root@controller ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens37
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens37"
DEVICE="ens37"
ONBOOT="yes"
IPADDR="10.10.10.180"
PREFIX="24"

```


- Trên compute1 config network & set hostname

`hostnamectl set-hostname compute1`

```
[root@compute1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="bd451710-01b3-4a7f-9d3c-83f5d25c9f07"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.239.181"
PREFIX="24"
GATEWAY="192.168.239.1"
DNS1="8.8.8.8"
```

```
[root@compute1 ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens37
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens37"
DEVICE="ens37"
ONBOOT="yes"
IPADDR="10.10.10.181"
PREFIX="24"

```

- Tắt firewall và selinux trên controller & compute1

```
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
init 6
```

### 2.1 Network Time Protocol (NTP)

- Chạy trên tất cả các node

`yum install chrony -y`

- Trên controller:

```
sed -i 's/server 0.centos.pool.ntp.org iburst/server 1.vn.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/server 0.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/server 3.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/server 3.centos.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/#allow 192.168.0.0\/16/allow 192.168.239.0\/24/g' /etc/chrony.conf
systemctl enable chronyd.service
systemctl start chronyd.service
chronyc sources
```

- Trên compute1:

```
sed -i 's/server 0.centos.pool.ntp.org iburst/server 192.168.239.180 iburst/g' /etc/chrony.conf
sed -i 's/server 1.centos.pool.ntp.org iburst/server 0.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 2.centos.pool.ntp.org iburst/server 3.asia.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/server 3.centos.pool.ntp.org iburst/server 1.vn.pool.ntp.org iburst/g' /etc/chrony.conf
sed -i 's/#allow 192.168.0.0\/16/allow 192.168.239.0\/24/g' /etc/chrony.conf
systemctl enable chronyd.service
systemctl start chronyd.service
chronyc sources
```

### 2.2 OpenStack packages

- Chạy trên tất cả các node

```
yum install centos-release-openstack-stein -y
yum upgrade
init 6
```

- Install the OpenStack client:

`yum install python-openstackclient -y`

`yum install openstack-selinux -y`

### 2.3 SQL database

`yum install mariadb mariadb-server python2-PyMySQL -y`

- Tạo file `/etc/my.cnf.d/openstack.cnf`

```
cat << EOF >> /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 192.168.239.180

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```

- Start the database service and configure it to start when the system boots:

```
systemctl enable mariadb.service
systemctl start mariadb.service
```

- Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

`mysql_secure_installation`


```
[root@controller ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
[root@controller ~]#

```

### 2.3 Message queue

- Install the package:

`yum install rabbitmq-server -y`

- Start the message queue service and configure it to start when the system boots:

```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

- Add the openstack user:

`rabbitmqctl add_user openstack Welcome123`

- Permit configuration, write, and read access for the openstack user:

`rabbitmqctl set_permissions openstack ".*" ".*" ".*"`


### 2.4 Memcached

- Install the packages:

`yum install memcached python-memcached -y`

```
sed -i "s/-l 127.0.0.1,::1/-l 192.168.239.180/g" /etc/sysconfig/memcached

systemctl enable memcached.service
systemctl restart memcached.service
```

### 2.5 Etcd

- Install the package:

`yum install etcd -y`

```
vi /etc/etcd/etcd.conf
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.239.180:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.239.180:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.239.180:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.239.180:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.239.180:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
```
```
systemctl enable etcd
systemctl start etcd
```

## 3. Minimal deployment for Stein


### 3.1 Keystone installation for Stein


- Tạo database:

```
mysql -u root -pWelcome123
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'Welcome123';
```

- Run the following command to install the packages:

`yum install openstack-keystone httpd mod_wsgi -y`

- Edit the /etc/keystone/keystone.conf

```
vi /etc/keystone/keystone.conf

[database]
connection = mysql+pymysql://keystone:Welcome123@192.168.239.180/keystone

[token]
provider = fernet

```

- Populate the Identity service database:

`su -s /bin/sh -c "keystone-manage db_sync" keystone`


```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

```
keystone-manage bootstrap --bootstrap-password Welcome123 \
  --bootstrap-admin-url http://192.168.239.180:5000/v3/ \
  --bootstrap-internal-url http://192.168.239.180:5000/v3/ \
  --bootstrap-public-url http://192.168.239.180:5000/v3/ \
  --bootstrap-region-id RegionOne

echo "ServerName controller" >> /etc/httpd/conf/httpd.conf  

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

systemctl enable httpd.service
systemctl start httpd.service
  
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://192.168.239.180:5000/v3
export OS_IDENTITY_API_VERSION=3

```

- Create a domain, projects, users, and roles



```
[root@controller ~]# openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 1c066743035b479faf48ce27a2d2340e |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+


[root@controller ~]# openstack project create --domain default \
>   --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | e4fc8eca3d7c40a98e5c0f5fa8a841ec |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+

[root@controller ~]# openstack project create --domain default \
>   --description "Demo Project" myproject
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 8eb79ab7b8b24855903ec903fa604d44 |
| is_domain   | False                            |
| name        | myproject                        |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+


[root@controller ~]# openstack user create --domain default \
>   --password Welcome123 myuser
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 22e0fa4837534d1e9aad67ca5165cbbf |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+


[root@controller ~]# openstack role create myrole
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 1038fb844e6d430a85ae2b66ef2e128d |
| name        | myrole                           |
+-------------+----------------------------------+

[root@controller ~]# openstack role add --project myproject --user myuser myrole

```

- As the admin user, request an authentication token:

```
openstack --os-auth-url http://192.168.239.180:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-20T19:02:30+0000                                                                                                                                                                |
| id         | gAAAAABc4uu2dOb_pyFVPiWTRsSx4ln93ILZR_9KrYPJlJmomYyKQWQ9KECAgnBnJR2vP8E1c7lV5fbzzI4jnMIn9tWJ4_5duLAFLuULLUwkBGTjuDSU--IdpLd8JEDnEbnMG9wDzHUGDmLDuyIonNafqxuuEbz1-EQHXDDRvOC1QzsciwZ6Hfw |
| project_id | cb881c79eb7d4fdc831135b6e5dab6d6                                                                                                                                                        |
| user_id    | 5facea8571cd447e8878d204397da2a4                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
  
  
  
openstack --os-auth-url http://192.168.239.180:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue  

+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-20T19:02:51+0000                                                                                                                                                                |
| id         | gAAAAABc4uvLahpGxZ2-lNXuRUhyhAUxs2MrXS8YGRsC3TlWcAxRqRQANRJ1Rze69oEHe3yPAF4gIQdg7IdSGzIuNG7yqbrQu0ZUEVlal6hkcSN4Do28-a2AOXEBYhHJvc7rWCmO5pDZCJOW6qwPhRaWL4XolDN_4eo3tJFItMCGIaAn2wMWKsg |
| project_id | 8eb79ab7b8b24855903ec903fa604d44                                                                                                                                                        |
| user_id    | 22e0fa4837534d1e9aad67ca5165cbbf                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
  
  

cat <<EOF>> admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://192.168.239.180:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

```


```
[root@controller ~]# . admin-openrc
[root@controller ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-20T19:03:21+0000                                                                                                                                                                |
| id         | gAAAAABc4uvp_WsvFZrbczIXKWP2oMbX0p9t4Q4QqaOOoe-pD5QFvUaAve4uacZ2I1pPzxx1R0tNfIUupdtbL-UtCsBHBH7Vnd2tfk9-RBad70rKEckdxNQ9ubreOCc_iAltBfIFi9DDyFgxdpvtf4rDkyBZJMf79gilcmuK0Za0Bj9O381UQiI |
| project_id | cb881c79eb7d4fdc831135b6e5dab6d6                                                                                                                                                        |
| user_id    | 5facea8571cd447e8878d204397da2a4                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
[root@controller ~]#
```

### 3.2 Glance installation for Stein

- Tạo database:

```
mysql -u root -pWelcome123

CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'Welcome123';
```

- Create the glance user:


```
openstack user create --domain default --password Welcome123 glance
openstack role add --project service --user glance admin
```

- Create the glance service entity:

```
openstack service create --name glance \
  --description "OpenStack Image" image
```

- Create the Image service API endpoints:

```
openstack endpoint create --region RegionOne \
  image public http://192.168.239.180:9292
openstack endpoint create --region RegionOne \
  image internal http://192.168.239.180:9292
openstack endpoint create --region RegionOne \
  image admin http://192.168.239.180:9292
```

- Install the packages:

`yum install openstack-glance wget -y`

- Edit the /etc/glance/glance-api.conf

```
[database]
connection = mysql+pymysql://glance:Welcome123@192.168.239.180/glance
[keystone_authtoken]
www_authenticate_uri  = http://192.168.239.180:5000
auth_url = http://192.168.239.180:5000
memcached_servers = 192.168.239.180:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

- Edit the /etc/glance/glance-registry.conf 

```
[database]
connection = mysql+pymysql://glance:Welcome123@192.168.239.180/glance

[keystone_authtoken]
www_authenticate_uri = http://192.168.239.180:5000
auth_url = http://192.168.239.180:5000
memcached_servers = 192.168.239.180:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
flavor = keystone
```

- Start the Image services and configure them to start when the system boots:


```
systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

`wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img`


```
openstack image create "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```

- Check

`openstack image list`  


```
[root@controller ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 059166f8-000e-444f-89bb-c61d8677eaad | cirros | active |
+--------------------------------------+--------+--------+
[root@controller ~]# openstack image show cirros
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                      |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                                                                                                                           |
| container_format | bare                                                                                                                                                                                       |
| created_at       | 2019-05-20T18:13:54Z                                                                                                                                                                       |
| disk_format      | qcow2                                                                                                                                                                                      |
| file             | /v2/images/059166f8-000e-444f-89bb-c61d8677eaad/file                                                                                                                                       |
| id               | 059166f8-000e-444f-89bb-c61d8677eaad                                                                                                                                                       |
| min_disk         | 0                                                                                                                                                                                          |
| min_ram          | 0                                                                                                                                                                                          |
| name             | cirros                                                                                                                                                                                     |
| owner            | cb881c79eb7d4fdc831135b6e5dab6d6                                                                                                                                                           |
| properties       | os_hash_algo='sha512', os_hash_value='6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e2161b5b5186106570c17a9e58b64dd39390617cd5a350f78', os_hidden='False' |
| protected        | False                                                                                                                                                                                      |
| schema           | /v2/schemas/image                                                                                                                                                                          |
| size             | 12716032                                                                                                                                                                                   |
| status           | active                                                                                                                                                                                     |
| tags             |                                                                                                                                                                                            |
| updated_at       | 2019-05-20T18:13:55Z                                                                                                                                                                       |
| virtual_size     | None                                                                                                                                                                                       |
| visibility       | public                                                                                                                                                                                     |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

### 3.3 Nova installation for Stein

### Node controller

- Tạo database

```
mysql -u root -pWelcome123

CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;


GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'Welcome123';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'Welcome123';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'Welcome123';
```  

- Create the Compute service credentials:

```
openstack user create --domain default --password Welcome123 nova

openstack role add --project service --user nova admin

openstack service create --name nova \
  --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne \
  compute public http://192.168.239.180:8774/v2.1

openstack endpoint create --region RegionOne \
  compute internal http://192.168.239.180:8774/v2.1

openstack endpoint create --region RegionOne \
  compute admin http://192.168.239.180:8774/v2.1
```

- Install the packages:

```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-novncproxy openstack-nova-scheduler -y`
```



