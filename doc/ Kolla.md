
##安装Kolla
### 环境依赖准备

1. 更新依赖

```
yum install epel-release
yum install python-pip
pip install -U pip
```

2. 安装依赖

```
yum install python-devel  libffi-devel gcc \
openssl-devel libselinux-python
```
3. 安装ansible

```
yum install ansible
```
4. 更新ansible 至最新版

```
pip install -U ansible
```

5. 添加配置 /etc/ansible/ansible.cfg
```
[defaults]
host_key_checking=False
pipelining=True
forks=100
```
### 安装Kolla-ansible
1. 安装kolla-ansible
```
pip install kolla-ansible
```
2. 复制 globals.yml及passwords.yml 至/etc/kolla directory
```
cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/

```

3. 复制 all-in-one 及 multinode 至当前文件夹
```
cp /usr/share/kolla-ansible/ansible/inventory/* .
```
### 安装Kolla

1. clone kolla 及 kolla-ansible
```
git clone https://github.com/openstack/kolla
git clone https://github.com/openstack/kolla-ansible
```

2. 安装 kolla和kolla-ansible依赖项
```
pip install -r kolla/requirements.txt
pip install -r kolla-ansible/requirements.txt
```
3. 将配置文件复制至 /etc/kolla
```
mkdir -p /etc/kolla
cp -r kolla-ansible/etc/kolla/* /etc/kolla
```

4. 将库存文件复制到当前目录。kolla-ansible在ansible/inventory目录中保存着库存文件(一体式和多联式)。

```
cp kolla-ansible/ansible/inventory/* .
```

## 准备安装
### 清单文件
下一步是准备我们的清单文件。库存是一个易于操作的文件，我们在其中指定节点角色和访问凭证。


Kolla-Ansible附带all-in-one和multinode的清单文件。它们之间的区别是前者可以在本地主机上部署单个节点OpenStack。如果你需要分离主机或使用多个节点，编辑multinode:

1. 编辑multinode的首个字段，其描述了环境的连接细节，例如：

```
[control]
10.0.0.[10:12] ansible_user=ubuntu ansible_password=foobar ansible_become=true

# Ansible supports syntax like [10:12] - that means 10, 11 and 12.
# Become clause means "use sudo".

[network:children]
control

# when you specify group_name:children, it will use contents of group specified.

[compute]
10.0.0.[13:14] ansible_user=ubuntu ansible_password=foobar ansible_become=true

[monitoring]
10.0.0.10

# This group is for monitoring node. 监控节点
# Fill it with one of the controllers' IP address or some others.

[storage:children]
compute

[deployment]
localhost       ansible_connection=local become=true

# use localhost and sudo
```

2. 检查multinode配置是否正确，使用命令

```
ansible -i multinode all -m ping
```
### Kolla密码
部署中使用的密码存储在/etc/kolla/Passwords.yml文件。所有密码在此文件是空白的，必须手动填写或运行随机密码发生器
对于部署环境运行:
```
kolla-genpwd
```


### Kolla globals.yml

globals.yml是Kolla-Ansible主要的配置文件.在部署Kolla-Ansible之前，有很多选项

* 镜像选择

用户需要选择其用于部署的镜像。用户必须指定要用于部署的映像。在本指南中，[DockerHub](ttps://hub.docker.com/u/kolla/)提供了预构建的映像。要了解更多关于构建机制的信息，请参考映像[构建文档](https://docs.openstack.org/kolla/latest/admin/image-building.html。)
其提供了多种镜像选择，包括多个不同的Linux 发行版本。
```
kolla_base_distro: "centos"
```
需要设置安装方式
    * binary：
  使用像apt或yum这样的存储库源
    * source：
  git存储库或本地源目录(此方式更为可靠)

```
kolla_install_type: "source"
```

使用DockerHub镜像，则默认镜像需要被重写，镜像以发布版本名称作为标签，例如，使用Pike镜像需要被设置为：
```
openstack_release: "pike"
```

使用与kolla-ansible相同版本的图像很重要。这意味着如果pip被用来安装kolla-ansible，这意味着它是最新的稳定版本，所以openstack版本应该设置为queens。如果git与master branch一起使用，DockerHub还提供了master branch的日常构建(标记为master):
```
openstack_release: "master"
```

* 网络设置

Kolla-Ansible需要设置一些网络选项。我们需要设置OpenStack使用的网络接口。需要设置的第一个接口是“network_interface”。这是multiple管理类型网络的默认接口。

参考文献：
[]()https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html
[]()http://xcodest.me/kolla-aio-install.html
