### 安装glance
glance为虚拟机提供虚拟机的镜像服务，其本身不负责实际的存储。
OpenStack 的镜像服务 (glance) 允许用户发现、注册和恢复虚拟机镜像。它提供了一个 REST API，允许您查询虚拟机镜像的 metadata 并恢复一个实际的镜像。您可以存储虚拟机镜像通过不同位置的镜像服务使其可用，就像 OpenStack 对象存储那样从简单的文件系统到对象存储系统。

* glance-api

    Accepts Image API calls for image discovery, retrieval, and storage.
* glance-registry

    Stores, processes, and retrieves metadata about images. Metadata includes items such as size and type.
* Database
    Stores image metadata and you can choose your database depending on your preference. Most deployments use MySQL or SQLite.
* Storage repository for image files
    Various repository types are supported including normal file systems (or any filesystem mounted on the glance-api controller node), Object Storage, RADOS block devices, VMware datastore, and HTTP. Note that some repositories will only support read-only usage.
    usage.
* Metadata definition service
    A common API for vendors, admins, services, and users to meaningfully define their own custom metadata. This metadata can be used on different types of resources like images, artifacts, volumes, flavors, and aggregates. A definition includes the new property’s key, description, constraints, and the resource types which it can be associated with.

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

* 创建glance用户

```
$ openstack user create --domain default --password-prompt glance

User Password:
Repeat User Password:
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

* 把admin角色添加到glance用户和service项目中
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

4. 创建glance服务的API endpoints

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
2. vim /etc/glance/glance-api.conf 文件进行如下操作：

* 在[database]字段, 设置数据库权限:
```
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
可将GLANCE_DBPASS更改为合适密码

* 在 [keystone_authtoken] 和[paste_deploy]字段,配置身份服务访问:
```
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
```
* 在 [glance_store] 字段, 设置本地文件系统及镜像存储位置
```
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

```
3. vim /etc/glance/glance-registry.conf 文件并完成以下操作

* 在 [database]字段, 设置数据库接入

```
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```

* 在 [keystone_authtoken] 及[paste_deploy] 字段, 设置用户服务

```
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
```
GLANCE_PASS 可以替换为可用的密码

4. 填充image服务数据库

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

#####　结束安装并配置

```
# systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service
# systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```
