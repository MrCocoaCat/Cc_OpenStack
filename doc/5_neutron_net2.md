***

#### 使用选项2 进行配置
* 安装组件

```
 yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

##### 配置server组件
vim /etc/neutron/neutron.conf 文件

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

* In the [oslo_concurrency] section, configure the lock path:

```
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

###### 配置 Modular Layer 2 (ML2) plug-in

 Modular Layer 2 (ML2) plug-in使用Linux桥接机制为实例构建第2层(桥接和交换)虚拟网络基础结构

vim /etc/neutron/plugins/ml2/ml2_conf.ini 文件

* 在ml2字段

```
[ml2]
# ...

#开启flat, VLAN, and VXLAN网络
type_drivers = flat,vlan,vxlan

#开启 VXLAN self-service 网络
tenant_network_types = vxlan

# 在 [ml2] 字段, 开启  Linux bridge 及 layer-2 population mechanisms:
mechanism_drivers = linuxbridge,l2population

# 在 [ml2]字段, 开启 port security extension driver
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

*vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini*

* 在 [linux_bridge] 字段, 将提供者虚拟网络映射到提供者物理网络接口
```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

```
 **将PROVIDER_INTERFACE_NAME 替换  为提供底层网络服务的物理网络端口名称**

> physical_interface_mappings = provider:ens3

* 在[vxlan]部分，启用vxlan覆盖网络，配置处理覆盖网络的物理网络接口的IP地址，并启用layer-2 population

```
[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
```
**将OVERLAY_INTERFACE_IP_ADDRESS替换为自己的IP地址**
>192.168.125.207

* 在 [securitygroup] 字段, enable security groups and configure the Linux bridge iptables firewall driver:
```
[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
* 确定  Linux 操作系统内核支持 network bridge filters by verifying all the following sysctl values are set to 1:

>vim /etc/sysctl.conf

```
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
###### 配置 layer-3 agent

*vim /etc/neutron/l3_agent.ini* 文件
* 在[DEFAULT] 字段, 配置 the Linux bridge interface driver and external network bridge:

```
[DEFAULT]
# ...
interface_driver = linuxbridge
```

###### 配置 DHCP agent

 DHCP agent为虚拟网络 提供 DHCP 服务
 *vim  /etc/neutron/dhcp_agent.ini* 文件，在[DEFAULT] 字段, 配置Linux桥接接口驱动程序Dnsmasq DHCP驱动程序，并启用隔离的元数据，以便提供者网络上的实例可以通过网络访问元数据
 ```
 [DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
 ```
