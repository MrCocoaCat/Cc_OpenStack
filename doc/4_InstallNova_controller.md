### Overview

OpenStack项目是一个开源的云计算平台，支持所有类型的云环境。该项目旨在实现简单的实现、大规模的可伸缩性和丰富的特性集。来自世界各地的云计算专家为这个项目做出了贡献。

OpenStack通过各种补充服务提供了一个基础设施即服务(IaaS)解决方案。每个服务都提供了一个应用程序编程接口(API)来促进这种集成。

本指南介绍了使用适合具有足够Linux经验的OpenStack新用户的功能示例体系结构逐步部署主要OpenStack服务的过程。本指南不打算用于生产系统安装，而是创建一个最小的概念验证，以学习OpenStack。

在熟悉了这些OpenStack服务的基本安装、配置、操作和故障排除之后，您应该考虑使用生产体系结构进行部署的以下步骤:
* 确定并实现必要的核心和可选服务，以满足性能和冗余需求。

* 使用防火墙、加密和服务策略等方法提高安全性。

* 实现部署工具，如Ansible、Chef、Puppet或Salt，以自动化生产环境的部署和管理
### Example architecture

示例体系结构需要至少两个节点(主机)才能启动基本虚拟机(VM)或实例。可选服务(如块存储和对象存储)需要额外的节点。

这个示例体系结构与最小生产体系结构的区别如下:

* 网络代理驻留在控制器节点上，而不是一个或多个专用网络节点。

* 用于自助网络的覆盖(隧道)流量通过管理网络而不是专用网络。

![](assets/markdown-img-paste-20180907221407513.png)

### Hardware requirements

https://docs.openstack.org/nova/queens/install/overview.html


### 安装Nova
Nova提供在統一的計算資源抽象

#####　安装必备条件

* 使用root帐号登录数据库
```
mysql -u root -p
```
* 创建nova_api, nova, 及 nova_cell0 数据库

```
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

```

* 配置权限

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
```
2. 获取admin用户的环境变量，并创建服务认证

```
. admin-openrc
```

3. 创建服务凭据

* 创建nova用户
```
$ openstack user create --domain default --password-prompt nova

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8a7dbf5279404537b1c7b86c033620fe |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```

* 把admin角色添加到nova用户和项目中
```
 openstack role add --project service --user nova admin
```
* 创建nova服务
```
$ openstack service create --name nova \
  --description "OpenStack Compute" compute

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 060d59eac51b4594815603d75a00aba2 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

4. 创建nova服务端点

```
$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 3c1caa473bfe4390a11e7177894bcc7b          |
| interface    | public                                    |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | e3c918de680746a586eac1f2d9bc10ab          |
| interface    | internal                                  |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 38f7af91666a47cfb97b4dc790b94424          |
| interface    | admin                                     |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 060d59eac51b4594815603d75a00aba2          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1               |
+--------------+-------------------------------------------+

```


5. 创建一个placement服务用户（Create a Placement service user using your chosen PLACEMENT_PASS:）

```
$ openstack user create --domain default --password-prompt placement

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fa742015a6494a949f67629884fc7ec8 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

6. 添加placement用户为placement服务admin角色

```
$ openstack role add --project service --user placement admin

```

7. 在服务目录创建Placement API

```
$ openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 2d1a27022e6e4185b86adac4444c495f |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

8. 创建Placement API服务端点

```
$ openstack endpoint create --region RegionOne placement public http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2b1b2637908b4137a9c2e0470487cbc0 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement internal http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 02bcda9a150a4bd7993ff4879df971ab |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne placement admin http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3d71177b9e0f406f98cbff198d74b182 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2d1a27022e6e4185b86adac4444c495f |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

#####　安装及配置组件

1. 安装软件包

```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api

```

2. vim /etc/nova/nova.conf文件

* 在[DEFAULT]字段中,设置API
```
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
```
* 在 [api_database]及[database]字段设置数据库接入许可

```
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
```

* 在[DEFAULT]字段, 设置RabbitMQ消息队列接入权限
```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
```
* 在[api]及[keystone_authtoken]字段中,设置身份验证服务
```
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS
```
* 在[DEFAULT]字段中,设置 my_ip 选项，为控制节点的IP地址

```
[DEFAULT]
# ...
my_ip = 192.168.125.115

```

* 在[DEFAULT]字段,开启网络服务

```
[DEFAULT]
# ...
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

```
* 在 [vnc] 字段中，设置  VNC代理为控制节点的IP

```
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

```

* 在[glance] 字段, 设置镜像服务API:

```
[glance]
# ...
api_servers = http://controller:9292
```

* 在[oslo_concurrency] 字段, 配置 lock 路径

```
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

```

* 由于软件包的一个bug，需要在/etc/httpd/conf.d/00-nova-placement-api.conf文件中添加如下配置
```
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```

* 重新http服务
```
systemctl restart httpd
```

3. 同步nova-api数据库

```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

>出现警告  /usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:332: NotSupportedWarning: Configuration option(s) ['use_tpool'] not supported
  exception.NotSupportedWarning
  将.py文件中332的语句注释掉


4. 注册cell0 数据库

```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

5. 创建cell1 cell

```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova 109e1d4b-536a-40d0-83c6-5f121b82b650
```

6. 同步nova数据库

```
su -s /bin/sh -c "nova-manage db sync" nova
```


7. 验证 nova、 cell0、 cell1数据库是否注册正确

```
# nova-manage cell_v2 list_cells
+-------+--------------------------------------+
| Name  | UUID                                 |
+-------+--------------------------------------+
| cell1 | 109e1d4b-536a-40d0-83c6-5f121b82b650 |
| cell0 | 00000000-0000-0000-0000-000000000000 |
+-------+--------------------------------------+
```
##### 完成安装

启动服务及至设置开机启动
```
# systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
  
# systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```
