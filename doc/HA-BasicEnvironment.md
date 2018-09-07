### 高可用性的硬件配置

OpenStack不需要大量的资源,具有核心服务和几个实例的高可用性环境应以下最低要求应支持

| Node type          | 核心数 |  内存 | 硬盘 | NIC|
| --------           | :---: |  :--:| :---:|:--:|
| controller node    | 4     | 12G  |120G  |2   |
| compute node       | 8+    | 12+G |120+G |2   |

我们建议任何两个控制器节点之间的最大延迟为2毫秒。虽然集群软件可以调优到更高的延迟，但一些供应商在同意支持安装之前坚持这个值。您可以使用ping命令查找两个服务器之间的延迟。

### 虚拟硬件设置

为了演示和学习，您可以在虚拟机(VMs)上设置测试环境。这有以下好处:
一个物理服务器可以支持多个节点，每个节点几乎支持任意数量的网络接口。您可以在整个安装过程中定期进行快照，并在出现问题时回滚到工作配置。但是，在vm上运行OpenStack环境会降低实例的性能，特别是当您的管理程序或处理器不支持嵌套vm的硬件加速时。

### 配置NTP

您必须配置NTP来正确地同步节点间的服务。我们建议您配置controller节点来引用更精确(底层)的服务器，并配置其他节点来引用controller节点。有关更多信息，请参阅安装指南。

### 安装Memcached

大多数OpenStack服务可以使用Memcached存储短暂的数据，比如令牌。尽管Memcached不支持典型形式的冗余(比如集群)，但OpenStack服务可以通过配置多个主机名或IP地址来使用几乎任意数量的实例。Memcached客户端实现了hash以平衡实例之间的对象。实例的失败只会影响某个对象的百分比，客户端会自动将其从实例列表中删除。内存缓存由osl.cache管理。

这确保了使用多个Memcached服务器时所有项目之间的一致性。下面是一个包含三个主机的配置示例:

Memcached_servers = controller1:11211 controller2:11211 controller3:11211

默认情况下，controller1处理缓存服务。如果主机宕机，controller2或controller3将完成服务。

### 配置共享服务

#### 高可用数据库

第一步是安装位于集群中心的数据库。要实现高可用性，请在每个控制器节点上运行数据库实例，并使用Galera集群在它们之间提供复制。Galera集群是基于MySQL和InnoDB存储引擎的同步多主数据库集群。它是一个高可用性服务，提供了高系统正常运行时间、无数据丢失和可伸缩性的增长。
根据您想要使用的数据库类型，可以通过许多不同的方式实现OpenStack数据库的高可用性。Galera集群有三种实现:
* Galera Cluster for MySQL
* MariaDB Galera Cluster: Galera集群的MariaDB实现，该集群通常在基于Red Hat发行版的环境中得到支持。
* Percona XtraDB Cluster
#### 高可用消息队列

为了协调进入系统的作业的执行，大多数OpenStack组件都需要一个符合AMQP(高级消息队列协议)的消息总线。
OpenStack安装中最流行的AMQP实现是RabbitMQ。
RabbitMQ节点在应用程序层和基础结构层上失败。应用层由多个AMQP主机的oslo配置选项控制。如果AMQP节点失败，应用程序将重新连接到指定的重新连接间隔内配置的下一个节点。
如果AMQP节点失败，应用程序将重新连接到指定的重新连接间隔内配置的下一个节点。

指定的重新连接间隔构成其SLA。
在基础架构层，SLA是RabbitMQ集群重新组装的时间。Mnesia管理器节点管理RabbitMQ相应资源。当它失败时，即为一个完整的AMQP集群停机时间间隔。通常，它的SLA不超过几分钟。
如果另一个节点是RabbitMQ的相应Pacemaker资源的从属节点，则完全不会导致AMQP集群停机。
使RabbitMQ服务高可用涉及以下步骤：
1. 安装RabbitMQ
2. 配置RabbitMQ为高可用队列
3. 配置OpenStack服务使用RabbitMQ高可用队列
##### 安装RabbitMQ

##### 配置RabbitMQ为高可用队列

以下服务或组件可以使用高可用队列
* OpenStack Compute
* OpenStack Block Storage
* OpenStack Networking
* Telemetry
考虑一下，虽然交换器和绑定可以在单个节点丢失的情况下存活下来，但是队列和它们的消息不会因为队列及其内容位于一个节点上而消失。如果丢失这个节点，也会丢失队列。RabbitMQ中的镜像队列提高了服务的可用性，因为它对故障具有弹性。生产服务器应该运行(至少)三个RabbitMQ服务器，以进行测试和演示，但是只能运行两个服务器。在本节中，我们配置了两个节点，称为rabbit1和rabbit2。确保所有节点都具有相同的Erlang cookie文件。

##### 配置OpenStack服务使用RabbitMQ高可用队列
配置Openstack组件，以确保其至少使用两个RabbitMQ节点
