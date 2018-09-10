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

>grant 权限1,权限2,…权限n on 数据库名称.表名称 to 用户名@用户地址 identified by ‘连接口令’;

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost'
IDENTIFIED BY 'KEYSTONE_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%'
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
2. vim /etc/keystone/keystone.conf 文件并完善以下字段

```
[database]
# 在[database]字段中, 设置数据库权限:
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
# 在 [token]字段中, 配置令牌提供者:
provider = fernet
```
>
将KEYSTONE_DBPASS 换成自己的database密码

3. 填充标识服务数据库

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
>su -s 指定执行的shell 即指定为/bin/sh





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

1. vim /etc/httpd/conf/httpd.conf 文件并且配置ServerName

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

####　创建用户

身份服务为每个OpenStack服务提供身份验证服务。

1. 创建默认domain


```
$ openstack domain create --description "An Example Domain" example

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 2f4f80574fd84fe6ba9067228ae0a50c |
| name        | example                          |
+-------------+----------------------------------+


```
2. 本指南使用一个服务项目，该项目为您添加到环境中的每个服务包含一个惟一的用户。创建创建service project

```
$ openstack project create --domain default \
  --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 24ac7f19cd944f4cba1d77469b2a73ed |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```

3. 常规(非管理)任务应该使用非特权项目和用户。作为示例，本指南创建演示项目和使用

* 创建demo project:

```
$ openstack project create --domain default \
  --description "Demo Project" demo

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
```

* Create the demo user:

```
$ openstack user create --domain default \
  --password-prompt demo

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aeda23aa78f44e859900e22c24817832 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```
> 0

* Create the user role:
```
$ openstack role create user

+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 997ce8d05fc143ac97d83fdfb5998552 |
| name      | user                             |
+-----------+----------------------------------+
```

* Add the user role to the demo project and user:

```

 openstack role add --project demo --user demo user
```


#### 验证操作 Verify operation

在安装其他服务之前，验证身份服务的操作。

1. 卸载临时OS_AUTH_URL和OS_PASSWORD环境变量:

```
unset OS_AUTH_URL OS_PASSWORD
```

2. 作为管理用户，请求一个身份验证令牌:

```
$ openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue

Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:14:07.056119Z                                     |
| id         | gAAAAABWvi7_B8kKQD9wdXac8MoZiQldmjEO643d-e_j-XXq9AmIegIbA7UHGPv |
|            | atnN21qtOMjCFWX7BReJEQnVOAj3nclRQgAYRsfSU_MrsuWb4EDtnjU7HEpoBb4 |
|            | o6ozsA_NmFWEpLeKy0uNn_WeKbAhYygrsmQGA49dclHVnz-OMVLiyM9ws       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+
```

>密码为： ADMIN_PASS
