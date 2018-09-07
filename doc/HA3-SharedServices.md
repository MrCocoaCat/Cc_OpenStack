## 配置共享服务

###高可用数据库

第一步是安装位于集群中心的数据库。要实现高可用性，请在每个控制器节点上运行数据库实例，并使用Galera集群在它们之间提供复制。Galera集群是基于MySQL和InnoDB存储引擎的同步多主数据库集群。它是一个高可用性服务，提供了高系统正常运行时间、无数据丢失和可伸缩性的增长。
[Galera详解](./Galera.md)

根据您想要使用的数据库类型，可以通过许多不同的方式实现OpenStack数据库的高可用性。Galera集群有三种实现:

* Galera Cluster for MySQL
* MariaDB Galera Cluster: Galera集群的 *MariaDB* 实现，该集群通常在基于Red Hat发行版的环境中得到支持。
* Percona XtraDB Cluster


### 高可用消息队列(RabbitMQ)

>息队列是一种应用间的异步协作机制

为了协调进入系统的作业的执行，大多数OpenStack组件都需要一个符合AMQP(高级消息队列协议)的消息总线。

>AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制

OpenStack安装中最流行的AMQP实现是 *RabbitMQ*。
RabbitMQ节点失效会发生在 *应用层（application）* 和 *基础结构层（infrastructure）* 上 。

*应用层(application)* 由多个AMQP主机的oslo.messaging配置选项控制。如果AMQP节点失败，应用层（application）将重新连接到指定的重新连接间隔内配置的下一个节点。
指定的重新连接间隔构成其SLA。

*基础架构层(infrastructure)*，SLA是RabbitMQ集群重新安装的时间。Mnesia管理器节点管理RabbitMQ相应资源。当它失败时，即为一个完整的AMQP集群停机时间间隔。通常，它的SLA不超过几分钟。
如果另一个节点是RabbitMQ的相应Pacemaker资源的从属节点，则完全不会导致AMQP集群停机。

使RabbitMQ服务高可用涉及以下步骤：
1. 安装RabbitMQ
2. 配置RabbitMQ为高可用队列
3. 配置OpenStack服务使用RabbitMQ高可用队列

##### 安装RabbitMQ
 略
##### 配置RabbitMQ为高可用队列

以下服务或组件可以使用高可用队列
* OpenStack Compute
* OpenStack Block Storage
* OpenStack Networking
* Telemetry
考虑一下，虽然交换器和绑定可以在单个节点丢失的情况下存活下来，但是队列和它们的消息不会因为队列及其内容位于一个节点上而消失。如果丢失这个节点，也会丢失队列。

RabbitMQ中的镜像队列提高了服务的可用性，因为它对故障具有弹性。生产服务器应该运行(至少)三个RabbitMQ服务器，以进行测试和演示，但是只能运行两个服务器。在本节中，我们配置了两个节点，称为rabbit1和rabbit2。确保所有节点都具有相同的Erlang cookie文件。

>为了解决上述各种问题，我们开发了 active/active HA queue 。从原理上讲，是 采用将 queue 镜像到 cluster 中的其他 node 上的方式实现 的。在该实现下，如果 cluster 中的一个 node 失效了，queue 能够自动地切换到镜像 queue 中的一个继续工作以保证服务的可用性。该解决方案仍然要求使用 RabbitMQ cluster ，这也意味着其无法在 cluster 内无缝地处理 network partition 问题。因此，不推荐跨 WAN 使用（虽然如此，客户端当然可以从远端或者近端进行连接）。

##### 配置OpenStack服务使用RabbitMQ高可用队列
配置Openstack组件，以确保其至少使用两个RabbitMQ节点

***
