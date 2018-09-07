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


>Memcached是一个自由开源的，高性能，分布式内存对象缓存系统.它通过在内存中缓存数据和对象来减少读取数据库的次数，从而提高动态、数据库驱动网站的速度。Memcached基于一个存储键/值对的hashmap。一般的使用目的是，通过缓存数据库查询结果，减少数据库访问次数，以提高动态Web应用的速度、提高可扩展性

![](assets/markdown-img-paste-20180907100352790.png)

大多数OpenStack服务可以使用Memcached存储临时的数据，比如tokens。尽管Memcached不支持典型形式的冗余(比如集群)，但OpenStack服务可以通过配置多个主机名或IP地址来使用几乎任意数量的实例。Memcached客户端实现了hash以平衡实例之间的对象。

>一致性哈希(Consistant Hash)是一种特殊的哈希算法，提供了这样的一个哈希表，当重新调整大小的时候，平均只有部分（k/n）key需要重新映射哈希槽，而不像传统哈希表那样几乎所有key需要需要重新映射哈希槽”[ConsistantHash](./ConsistantHash)



[Memcached的github地址](https://github.com/Memcached/Memcached/wiki#getting-started)

内存缓存由osl.cache管理,其确保了使用多个Memcached服务器时所有项目之间的一致性。下面是一个包含三个主机的配置示例:

Memcached_servers = controller1:11211 controller2:11211 controller3:11211

默认情况下，controller1担任缓存任务。如果主机宕机，controller2或controller3将完成服务。
