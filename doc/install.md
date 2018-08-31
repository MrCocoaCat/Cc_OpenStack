利用Devstack 可以安装快速openstack，但为了更好的理清openstack，本文使用手动安装。
参考[](https://docs.openstack.org/install-guide/)
安装版本为**Queue** 版本，最小的openstack需安装以下组件：

*   Identity service – [keystone installation for Queens](https://docs.openstack.org/keystone/queens/install/)
*   Image service – [glance installation for Queens](https://docs.openstack.org/glance/queens/install/)
*   Compute service – [nova installation for Queens](https://docs.openstack.org/nova/queens/install/)
*   Networking service – [neutron installation for Queens](https://docs.openstack.org/neutron/queens/install/)

建议安装的软件包为：

*   Dashboard – [horizon installation for Queens](https://docs.openstack.org/horizon/queens/install/)
*   Block Storage service – [cinder installation for Queens](https://docs.openstack.org/cinder/queens/install/)
### 安装环境

#### 设置主机名及IP

1. 编辑 /etc/hosts

```
IP   controller
```
即 "IP地址，域名，主机名"
其中域名可以省略，不要删除127.0.0.1项。
> *192.168.125.115   controller*

#### 网络时间同步协议(NTP)
1. 安装包
```
yum install chrony
```

2. 编辑chrony.conf文件
在/etc/chrony.conf文件中写入以下内容

```
server NTP_SERVER iburst
```
NTP_SERVER 为主机名或IP地址
> *server 192.168.125.115 iburst*

3. 保证其他服务节点可以访问控制节点的chrony daemon,需要在同一个chrony.conf文件中写入以下内容

```
allow 10.0.0.0/24
```
将10.0.0.0/24　替换为相对的子网
> *allow 192.168.125.0/24*

４．重启NT服务

```
systemctl enable chronyd.service
systemctl start chronyd.service
```

#### SQL 数据库
大多数OpenStack服务使用SQL数据库来存储信息。数据库通常在controller节点上运行。本指南中的过程根据发行版使用MariaDB或MySQL。

1. 安装相应软件包

```
yum install mariadb mariadb-server python2-PyMySQL
```
2. 创建并修改/etc/my.cnf.d/openstack.cnf文件，并新增[mysql]字段，并将bind-address设置为管理节点地址，并将编码方式设置未utf8,修改内容如下

```
[mysqld]
# 监听地址,0.0.0.0设置为全部可以监听
# 可以设置未controller的IP地址
bind-address = 0.0.0.0

# 默认存储引擎innodb
default-storage-engine = innodb

# 设置独享的表空间，如果不设置，会是共享表空间
innodb_file_per_table = on

# 最大连接数
max_connections = 4096

# 校对规则
collation-server = utf8_general_ci

# 数据库建库字符集
character-set-server = utf8
```


3. 启动服务，并设置为开机启动
```
＃ 设置为开机启动
systemctl enable mariadb.service
＃　启动服务
systemctl start mariadb.service

```

4. 执行mysql_secure_installation脚本设置安全属性，并为root帐号设置合适密码
```
mysql_secure_installation
```

[SQL database](https://docs.openstack.org/install-guide/environment-sql-database.html)

#### 安装openstack相关包
1. 安装包
```
yum install centos-release-openstack-queens
```

2. 更新软件包
```
yum upgrade
```

3. 安装OpenStack client
```
yum install python-openstackclient
```

4. RHEL and CentOS 默认启动了 SELinux 安装openstack-selinux为openstack服务器自动管理安全策略
```
yum install openstack-selinux
```
####　消息队列RabbitMQ
OpenStack使用消息队列来协调服务之间的操作和状态信息。
消息队列服务通常在**控制器节点**上运行。OpenStack支持多个消息队列服务，包括RabbitMQ、Qpid和ZeroMQ。但是，大多数penStack的发行版都支持特定的消息队列服务。因为大多数发行版均支持RabbitMQ消息队列服务，故安装RabbitMQ消息队列
1. 安装包
```
yum install rabbitmq-server
```

2. 启动消息队列并设置为开机启动
```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```
3. 添加openstack用户
```
rabbitmqctl add_user openstack RABBIT_PASS
```
RABBIT_PASS 替换为合适的密码

4. 许可设定，未openstack用户添加读写权限
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 安装Memcached
服务的身份服务身份验证机制使用Memcached缓存令牌。memcached服务通常在**控制器节点**上运行。

1. 安装相应软件包
```
yum install memcached python-memcached
```

2. 编写文件/etc/sysconfig/memcached
确定服务使用的是controller节点的management　IP地址,以使其他节点可以通过management网络访问控制节点

```
OPTIONS="-l 127.0.0.1,::1,controller"
```
>更改之前的OPTIONS="-l 127.0.0.1,::1".


3. 开启Memcached服务并设为开机启动

```
systemctl enable memcached.service
systemctl start memcached.service
```
#### Etcd
OpenStack服务可以使用Etcd，这是一种分布式可靠的键值存储，用于分布式键锁定、存储配置、跟踪服务实时性和其他场景。
1. 安装相应软件包
```
yum install etcd
```
2. 编写/etc/etcd/etcd.conf 文件并设置以下字段  ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS．使其他节点可以连接值管理网络
```
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.125.115:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.125.115:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.125.115:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.125.115:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.125.115:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
```
其中10.0.0.11为控制节点的网络，需改成自己的IP地址

3. 启动服务
```
systemctl enable etcd
systemctl start etcd
```

### 安装keysnote（Identity service）

##### 在安装和配置认证服务之前，首先确保已创建数据库
1. 使用root帐号连接数据库
```
mysql -u root -p
```
2. 创建keystone数据库
```
MariaDB [(none)]> CREATE DATABASE keystone;
```
3. 进行授权
```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```
可将其中的KEYSTONE_DBPASS替换为合适的密码
使用如下命令可以差数据库状态
```
MariaDB [(none)]> show databases;
```

##### 安装和配置组件
1. 安装包
```
yum install openstack-keystone httpd mod_wsgi
```
2. 编写 /etc/keystone/keystone.conf 文件并完善以下字段

* 在[database]字段中, 设置数据库权限:
```
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
```
>
将KEYSTONE_DBPASS 换成自己的database密码
* 在 [token]字段中, 配置令牌提供者:
```
[token]
# ...
provider = fernet
```
3. 填充标识服务数据库
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
4. 初始化Fernet密钥存储库

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

```
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
5. 引导标识服务
```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

```
将ADMIN_PASS替换为合适的密码
##### 配置Apache HTTP 服务

1. 编辑 /etc/httpd/conf/httpd.conf 文件并且配置ServerName

```
ServerName controller
```
2. 为 /usr/share/keystone/wsgi-keystone.conf 文件创建链接
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

##### 安装完成启动服务
1. 启动 Apache HTTP 服务并设置为开机启动

```
systemctl enable httpd.service
systemctl start httpd.service
```
2. 配置管理账户

```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=ADMIN_PASS
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://controller:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

#####　创建用户
身份服务为每个OpenStack服务提供身份验证服务。
1. 创建默认domain


```
$ openstack domain create --description "Default Domain" default

```
2. 为您的环境中的管理操作创建一个管理项目、用户和角色

* 创建admin project

```
$ openstack project create --domain default \
  --description "Admin Project" admin
```

* 创建admin user

```
$ openstack user create --domain default \
  --password-prompt admin
```

* 创建admin role





### 安装glance
glance为虚拟机提供虚拟机的镜像服务，其本身不负责实际的存储
#####　安装必备条件

1. 创建数据库
使用root帐号登录数据库
```
mysql -u root -p
```
创建glance数据库
```
MariaDB [(none)]> CREATE DATABASE glance;
```
为glance数据库赋予权限
```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';

```
2. 获取admin用户的环境变量，并创建服务认证

```
. admin-openrc
```

3. 要创建服务凭据

* 创建glance user

```
openstack user create --domain default --password-prompt glance
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f4e777c4062483ab8d9edd7dff829df |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

* 把admin用户添加到glance用户和项目中
```
openstack role add --project service --user glance admin
```
> 此命令无返回值

* 创建glance服务
```
$ openstack service create --name glance \
  --description "OpenStack Image" image

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

4. 创建镜像服务API端点

```
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 340be3625e9b4239a6415d034e98aace |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a6e4b153c2ae4c919eccfdbb7dceb5d2 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0c37ed58103f4300a84ff125a539032d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

```

#####　安装和配置组件
1. 安装软件包
```
yum install openstack-glance
```
2. Edit the /etc/glance/glance-api.conf file and complete the following actions: