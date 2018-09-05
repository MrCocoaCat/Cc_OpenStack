
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

[]()https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html
[]()http://xcodest.me/kolla-aio-install.html
