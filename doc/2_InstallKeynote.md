### 安装keysnote（Identity service）

##### 在安装和配置认证服务之前，首先确保已创建数据库
1. 使用root帐号连接数据库
```
mysql -u root -p
```
2. 创建keystone数据库
```
MariaDB [(none)]> CREATE DATABASE keystone;
```
3. 进行授权
```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```
可将其中的KEYSTONE_DBPASS替换为合适的密码
使用如下命令可以查看数据库状态
```
MariaDB [(none)]> show databases;
```

##### 安装和配置组件
1. 安装包
```
yum install openstack-keystone httpd mod_wsgi
```
2. 编写 /etc/keystone/keystone.conf 文件并完善以下字段

* 在[database]字段中, 设置数据库权限:
```
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
```
>
将KEYSTONE_DBPASS 换成自己的database密码
* 在 [token]字段中, 配置令牌提供者:
```
[token]
# ...
provider = fernet
```
3. 填充标识服务数据库
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
4. 初始化Fernet密钥存储库

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

```
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
5. 引导标识服务
```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

```
将ADMIN_PASS替换为合适的密码

##### 配置Apache HTTP 服务

1. 编辑 /etc/httpd/conf/httpd.conf 文件并且配置ServerName

```
ServerName controller
```
2. 为 /usr/share/keystone/wsgi-keystone.conf 文件创建链接
```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

##### 安装完成启动服务
1. 启动 Apache HTTP 服务并设置为开机启动

```
systemctl enable httpd.service
systemctl start httpd.service
```
2. 配置管理账户

```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=ADMIN_PASS
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://controller:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```

#####　创建用户
身份服务为每个OpenStack服务提供身份验证服务。
1. 创建默认domain


```
$ openstack domain create --description "Default Domain" default

```
2. 为您的环境中的管理操作创建一个管理项目、用户和角色

* 创建admin project

```
$ openstack project create --domain default \
  --description "Admin Project" admin
```

* 创建admin user

```
$ openstack user create --domain default \
  --password-prompt admin
```

* 创建admin role
