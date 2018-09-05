[]()https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html

[]()http://xcodest.me/kolla-aio-install.html

本指南逐步提供了使用Kolla在裸金属服务器或虚拟机上部署OpenStack的说明。根据操作人员的确切要求进行定量。

在下面的内容中，我们以 Kolla & Kolla-ansible 的组合，主要分 3 步介绍如何轻松部署出一套 OpenStack 环境。

1. 环境依赖准备
2. 使用 Kolla 构建镜像
使用
3. Kolla-ansible 部署 OpenStack

### 环境依赖准备

1. 更新依赖

```
yum install epel-release
yum install python-pip
pip install -U pip
```

2. 安装依赖

```
yum install python-devel  libffi-devel gcc openssl-devel libselinux-python
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
1. 
