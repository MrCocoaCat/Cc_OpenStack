[]()https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html

[]()http://xcodest.me/kolla-aio-install.html

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
