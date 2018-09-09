


本节描述如何在控制器节点上安装和配置代码为cinder的块存储服务。
此服务至少需要一个额外的存储节点，该节点为实例提供卷。
### Prerequisites
1. 创建数据库

a.登入数据库
```
mysql -u root -p
```
b. 创建cinder 数据库

```
MariaDB [(none)]>
```

c. 为cinder 创建合适的权限

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'CINDER_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'CINDER_DBPASS';

```

2.

```
$ . admin-openrc
```

3. 创建 the service credentials

a. 创建cinder user
```
$ openstack user create --domain default --password-prompt cinder

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 9d7e33de3e1a498390353819bc7d245d |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

b. 为conder用户添加admin角色:

```
$ openstack role add --project service --user cinder admin
```
c. Create the cinderv2 and cinderv3 service entities:

```
$ openstack service create --name cinderv2 \
 --description "OpenStack Block Storage" volumev2

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | eb9fd245bdbc414695952e93f29fe3ac |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+

$ openstack service create --name cinderv3 \
 --description "OpenStack Block Storage" volumev3

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | ab3bbbef780845a1a283490d281e7fda |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+


```

4. Create the Block Storage service API endpoints:

```
$ openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 513e73819e14460fb904163f41ef3759         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

$ openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 6436a8a23d014cfdb69c586eff146a32         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

$ openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | e652cf84dd334f359ae9b045a2c91d96         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+

$ openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 03fa2c90153546c295bf30ca86b1344b         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | ab3bbbef780845a1a283490d281e7fda         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

$ openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 94f684395d1b41068c70e4ecb11364b2         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | ab3bbbef780845a1a283490d281e7fda         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

$ openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 4511c28a0f9840c78bacb25f10f62c98         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | ab3bbbef780845a1a283490d281e7fda         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+


```


### 安装组件

1. 安装组件

```
yum install openstack-cinder
```

2. vim  the /etc/cinder/cinder.conf

a.  configure database access:

```
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

```

a. configure RabbitMQ message queue access:

```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

```


b. In the [DEFAULT] and [keystone_authtoken] sections, configure Identity service access:

```
[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```

c. In the [DEFAULT] section, configure the my_ip option to use the management interface IP address of the controller node:

```
[DEFAULT]
# ...
my_ip = 192.168.125.207
```

3. In the [oslo_concurrency] section, configure the lock path:

```
[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp

```

4. Populate the Block Storage database:

```
 su -s /bin/sh -c "cinder-manage db sync" cinder
```

### Configure Compute to use Block Storag

1.  vim /etc/nova/nova.conf

```
[cinder]
os_region_name = RegionOne

```

### Finalize installation

```
# systemctl restart openstack-nova-api.service

# systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service

# systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```
