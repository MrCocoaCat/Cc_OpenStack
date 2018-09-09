### 安装dashboard
 本节描述如何在控制器节点上安装和配置仪表板。仪表板惟一需要的核心服务是标识服务。您可以将仪表板与其他服务(如映像服务、计算和联网)结合使用。您还可以在带有对象存储等独立服务的环境中使用仪表板。

#####　安装必备条件

1. 安装软件包
```
yum install openstack-dashboard
```

2. vim  /etc/openstack-dashboard/local_settings 文件

* 配置dashboard以使用OpenStack服务，在控制节点上

```
OPENSTACK_HOST = "controller"
```

* 使hosts可以访问
```
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']
```
>设置中[ ‘ * ’ ] 使所用均可以访问

* 配置 **memcached** session storage service

```
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

```
* 开启Identity API version 3
```
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
```

* Enable support for domains:

```
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```


* 配置API版本

```
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
```

* 通过dashboard创建的默认类型

```
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

```

* 设置创建的默认角色

```
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

```


* 如果选择 **networking option 1**, disable support for layer-3 networking services:
```
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

```

* Optionally, configure the time zone:
```
TIME_ZONE = "TIME_ZONE"
```

3. 配置
/etc/httpd/conf.d/openstack-dashboard.conf
加入以下内容
```
WSGIApplicationGroup %{GLOBAL}
```
#####　开启服务

```
systemctl restart httpd.service memcached.service
```
