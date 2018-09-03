### 安装neutron

#####　安装必备条件

1. 创建数据库
* 登录数据库
```
mysql -u root -p
```
* 创建neutron数据库
```
MariaDB [(none)] CREATE DATABASE neutron;
```

* 设置登录数据库的权限

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'NEUTRON_DBPASS';

```

2. 获取admin权限

```
. admin-openrc
```

3. 创建服务凭证

* 创建neutron用户:

```
$ openstack user create --domain default --password-prompt neutron

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fdb0f541e28141719b6a43c8944bf1fb |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```
* 添加admin角色为neutron用户

```
 openstack role add --project service --user neutron admin
```

* 创建neutron服务

```
$ openstack service create --name neutron \
  --description "OpenStack Networking" network

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | f71529314dab4a4d8eca427e701d209e |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

4. 创建网络服务API端点

```
$ openstack endpoint create --region RegionOne \
  network public http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 85d80a6d02fc4b7683f611d7fc1493a3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  network internal http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 09753b537ac74422a68d2d791cf3714f |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  network admin http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 1ee14289c9374dffb5db92a5c112fc4e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f71529314dab4a4d8eca427e701d209e |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```
5. 配置网络选项

可以使用选项1和2表示的两种体系结构之一来部署网络服务。
选项1：部署最简单的体系结构，只支持将实例附加到提供者(外部)网络。
没有自助(私有)网络、路由器或浮动IP地址。只有管理员或其他特权用户可以管理提供者网络。
选项2：选项2在选项1中增加了支持将实例附加到自助服务网络的第3层服务。
demo程序或其他非特权用户可以管理自助服务网络，包括提供自助服务和提供者网络之间连接的路由器。另外，浮动IP地址通过外部网络(如Internet)提供到使用自助服务网络的实例的连接。

Self-service网络通常使用覆盖网络。覆盖网络协议(如VXLAN)包括额外的头文件，增加开销并减少有效负载或用户数据的可用空间。在不了解虚拟网络基础结构的情况下，实例尝试使用默认的以太网最大传输单元(MTU)发送数据包，其大小为1500字节。
网络服务通过DHCP自动为实例提供正确的MTU值。但是，有些云映像不使用DHCP或忽略DHCP MTU选项，需要使用元数据或脚本进行配置。

#####使用选项2 进行配置
* 安装组件
```
 yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

###### 配置服务组件
编辑/etc/neutron/neutron.conf 文件
  * 配置数据库权限，在[database]字段写入以下内容
```
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

```
* 在[DEFAULT]字段, 开启Modular Layer 2 (ML2) plug-in, router service, and overlapping IP addresses:

```
[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

```
* 在[DEFAULT]字段,配置RabbitMQ权限

```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
```

* 在[DEFAULT]及[keystone_authtoken]字段, 配置认证服务

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
username = neutron
password = NEUTRON_PASS
```

* 在[DEFAULT]和[nova]部分中，配置连网以通知计算网络拓扑变化:

```
[DEFAULT]
# ...
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
# ...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
```

###### 配置 Modular Layer 2 (ML2) plug-in

 Modular Layer 2 (ML2) plug-in使用Linux桥接机制为实例构建第2层(桥接和交换)虚拟网络基础结构
编辑 /etc/neutron/plugins/ml2/ml2_conf.ini 文件
* 在ml2字段开启flat, VLAN, and VXLAN网络
```
[ml2]
# ...
type_drivers = flat,vlan,vxlan

```
* 开启 VXLAN self-service 网络
```
[ml2]
# ...
tenant_network_types = vxlan
```
* 在 [ml2] 字段, 开启  Linux bridge 及 layer-2 population mechanisms:
```
[ml2]
# ...
mechanism_drivers = linuxbridge,l2population
```

* 在 [ml2]字段, 开启 port security extension driver
```
[ml2]
# ...
extension_drivers = port_security
```

* 在 [ml2_type_flat] 字段, 配置虚拟机网络为flat network
```
[ml2_type_flat]
# ...
flat_networks = provider
```
* 在 [ml2_type_vxlan] 字段, 配置 self-service networks的VXLAN network identifier范围
```
[ml2_type_vxlan]
# ...
vni_ranges = 1:1000
```
* 在[securitygroup] 字段, enable ipset to increase efficiency of security group rules
```
[securitygroup]
# ...
enable_ipset = true
```

###### 配置 Linux bridge agent

Linux bridge agent为实例构建第2层(桥接和交换)虚拟网络基础设施，并处理安全组。
编辑 /etc/neutron/plugins/ml2/linuxbridge_agent.ini 文件

* 在 [linux_bridge] 字段, 将提供者虚拟网络映射到提供者物理网络接口
```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

```
 将PROVIDER_INTERFACE_NAME 替换  为提供底层网络服务的物理网络端口名称

> physical_interface_mappings = provider:ens3

* 在[vxlan]部分，启用vxlan覆盖网络，配置处理覆盖网络的物理网络接口的IP地址，并启用layer-2 population

```
[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
```
将OVERLAY_INTERFACE_IP_ADDRESS替换为自己的IP地址

* 在 [securitygroup] 字段, enable security groups and configure the Linux bridge iptables firewall driver:
```
[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
* 确定  Linux 操作系统内核支持 network bridge filters by verifying all the following sysctl values are set to 1:
```
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
```
###### 配置 layer-3 agent
编辑 /etc/neutron/l3_agent.ini 文件
* 在[DEFAULT] 字段, 配置 the Linux bridge interface driver and external network bridge:

```
[DEFAULT]
# ...
interface_driver = linuxbridge
```
