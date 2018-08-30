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

#### Network Time Protocol (NTP)
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

3. 保证其他服务节点可以访问控制节点的chrony daemon,需要在同一个chrony.conf文件中写入以下内容

```
allow 10.0.0.0/24
```

将10.0.0.0/24　替换为相对的子网

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
2. 创建并修改/etc/my.cnf.d/openstack.cnf文件，并添加如下内容
```
[mysqld]
# 监听地址,0.0.0.0设置为全部可以监听
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
新增[mysql]字段，并将bind-address设置为管理节点地址，并将编码方式设置未utf8

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

#### 安装openstack包
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

### 安装Identity service
