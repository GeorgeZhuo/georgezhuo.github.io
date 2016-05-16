---
layout: post
title: Install radosgw on ubuntu
tags: ceph

---

# INSTALL-RADOSGW ON UBUNTU 14.04

``` 
我的环境是：  
Ubuntu 14.04 (Debian)  
如果是RPM的话，安装方法不同，需要参考如下。  
![ceph](http://docs.ceph.com/docs/master/install/install-ceph-gateway/)
``` 

由于Apache的版本不同，配置FASTCGI的时候是不一样的，所以需要注意一下。  
在某些版本安装Apache2.4(如 RHEL 7, CentOS 7, 或者Ubuntu 14.04 Trusty)  
这些版本的Linux系统，mod_proxy_fcgi是已经有的。所以当安装了Apache  
或者httpd(RPM-based), mod_proxy_fcgi 就已经可以使用的了。  
而在Apache2.2版本的中(如 RHEL 6， CentOS 6，Ubuntu 12.04 Precise)  
mod_proxy_fcgi 是另外的安装包，在 RHEL6/CentOS6, 在EPEL6 版本中  
可以使用 yum install mod_proxy_fcgi. Ubuntu 12.04, 似乎又问题。  
![详情见](https://bugs.launchpad.net/precise-backports/+bug/1422417)

## 安装Apache

### 安装Apache2  
**sudo apt-get install apache**

### 配置Apache
Apache 的配置文件在/etc/apache2/apache2.conf, 需要添加ServerName  
即主机的全称域名FQDN.(eg, hostname -f).

```

``` 

加mod_proxy_fcgi模块  
**sudo a2enmod proxy_fcgi**

然后启动Apache  
**sudo service apache2 start**

### 启动SSL
有些 REST 客户端默认使用HTTPS. 所以需要考虑为Apache启动SSL。  
安装相关依赖：
**sudo apt-get install openssl ssl-cert**

启动SSL模块:  
**sudo a2enmod ssl**

生成证书：
sudo mkdir /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout \  
/etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt

重启Apache：  
sudo service apache2 restart

## 安装Radosgw
安装 Radosgw:  
**sudo apt-get install radosgw**

## 配置Radosgw
要配置Ceph对象网关要求要安装好Ceph集群和带有FastCGI模块Apache。  
Ceph对象网关相当于Ceph集群的客户端。作为Ceph的客户端需要如下：  
1. 一个名字作为网关实例。 (我用gateway)  
2. 一个在keyring中拥有一定权限的存储集群的用户名。  
3. Ceph集群中有存储数据的资源池 Pool  
4. 网关实例的数据目录  
5. Ceph 配置文件中有实例的相关信息   
6. 一个用于Apache和FastCGI交互的配置文件

### 创建用户和Keyring
每一个实例必须要有一个用户名和key才可以访问Ceph集群。  
在下来的步骤中，我们用管理节点来创建key。  
* 首先是创建一个客户端用户名和key。  
* 然后将key加入到集群中。  
* 最后我们将key ring 放到网关实例所在的节点。  

如果是使用docker部署ceph的话，需要在下面使用ceph-authool, ceph auth,  
ceph 前面加上*docker exec mon ceph*

#### 为网关gateway创建keyring  
```
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
sudo chmod +r /etc/ceph/ceph.client.radosgw.keyring
```
需要为keyring添加可读权限，启动radosgw时，可能会报错。  

#### 为每一个实例生成对象网关的用户名和key. 我们网关的名字为gateway。  
所以client.radosgw跟就是gateway.  
``` 
sudo ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.gateway --gen-key
```

#### key添加权限能力。   
```
sudo ceph-authtool -n client.radosgw.gateway --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
```
#### 需要是创建的keyring和key能够让对象网关可以访问Ceph集群，
需要将key加入到Ceph集群中。
```
sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway -i /etc/ceph/ceph.client.radosgw.keyring
``` 
#### 如果管理节点不是网关所在的主机，需要将keyring放到对应的目录下  
如果在同一个主机上就不用了。  

#### 查看是将添加成功.  
**ceph auth list**  

### 创建Pool
**ceph osd pool create {pool_name} {pg_num} {pgp_num}**

* .rgw.root
* .rgw.control
* .rgw.gc
* .rgw.buckets
* .rgw.buckets.index
* .rgw.buckets.extra
* .log
* .intent-log
* .usage
* .users
* .users.email
* .users.swift
* .users.uid

查看是否创建成功。  
**ceph osd lspools**   

### 在Ceph配置中添加网关
在管理节点为Ceph的配置文件添加对象网关的配置。对象网关的配置要求你区别对象网关的实例。  
在安装对象网关deamon时需要指定主机名，keyring, 为FastCGI的socket路径，日志文件。  
不同Apache版本配置时参数是不一样的。我的环境是Ubuntu 14.04, Apache 2.4.7.  
* 在Apache2.2和Apache2.4的早期版本(RHEL 6, Ubuntu 12.04, 14.04), 在/etc/ceph/ceph.conf添加：  
```
[client.radosgw.gateway]
host = {hostname}
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = ""
log file = /var/log/radosgw/client.radosgw.gateway.log
rgw frontends = fastcgi socket_port=9000 socket_host=0.0.0.0
rgw print continue = false
```
**注:**
**Apache2.2和Apache2.4的早期版本不是使用Unix Domain Sockets而是localhost TCP。**   

下面是针对于其他版本下的，主要的区别就是支持UDS, 所以配置文件不同。  
在Apache2.4.9后的版本(RHEL 7, CentOS7)，在/etc/ceph/ceph.conf添加:  
```
[client.radosgw.gateway]
host = {hostname}
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock
log file = /var/log/radosgw/client.radosgw.gateway.log
rgw print continue = false
```
**注:**
**Apache2.4.9支持Unix Domain Socket(UDS), 但是Ubuntu 14.04　安装的是Apache2.4.7.   
所以不支持UDS, 而是配置为localhost TCP.**  
{hostname} 是提供网关服务的短的主机域名(hostname -s 的输出)。  

由于Ceph中默认的日志级别不高，所以可以在ceph.conf中添加:  
```
[global]
debug ms = 1
debug rgw = 20
```
但是这样日志量会有点大, 如果是docker部署的话，docker的日志现在没有支持清除。  

### 创建数据目录
配置文件默认不用创建网关的数据目录，所以需要手动创建。  
```
sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.gateway
```

### 启动Radosgw
**sudo /etc/init.d/radosgw start**

启动失败可以，在/var/log/radosgw/client.radosgw.gateaway.log查看日志。  


### 添加网关配置文件
Apache版本不同和Linux系统不同，配置文件不同，要注意区别。  
Ubuntu是在/etc/apache2/conf-available, 为对象网关创建rgw.conf.  
* 在Apache2.2和Apache2.4的早期版本(RHEL 6, Ubuntu 12.04, 14.04), 我的环境是Ubuntu 14.04,  
所以添加：  

```
<VirtualHost *:80>
ServerName localhost
DocumentRoot /var/www/html
ErrorLog /var/log/httpd/rgw_error.log
CustomLog /var/log/httpd/rgw_access.log combined
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]
SetEnv proxy-nokeepalive 1
ProxyPass / fcgi://localhost:9000/
</VirtualHost>
```

下面是针对于其他版本下的，主要的区别就是支持UDS, 所以配置文件不同。  
在Apache2.4.9后的版本(RHEL 7, CentOS7)添加:

```
<VirtualHost *:80>
ServerName localhost
DocumentRoot /var/www/html
ErrorLog /var/log/httpd/rgw_error.log
CustomLog /var/log/httpd/rgw_access.log combined
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]
SetEnv proxy-nokeepalive 1
ProxyPass / unix:///var/run/ceph/ceph.radosgw.gateway.fastcgi.sock|fcgi://localhost:9000/
</VirtualHost>
```

重启Apache:  
**sudo service apache2 restart **   

## 使用网关

### 为S3访问创建网关用户　
　
在网关主机上执行如下命令:  
```
sudo radosgw-admin user create --uid="testuser" --display-name="First User"
```
输出的结果如下：
```
{"user_id": "testuser",
"display_name": "First User",
"email": "",
"suspended": 0,
"max_buckets": 1000,
"auid": 0,
"subusers": [],
"keys": [
{ "user": "testuser",
"access_key": "I0PJDPCIYZ665MW88W9R",
"secret_key": "dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA"}],
"swift_keys": [],
"caps": [],
"op_mask": "read, write, delete",
"default_placement": "",
"placement_tags": [],
"bucket_quota": { "enabled": false,
"max_size_kb": -1,
"max_objects": -1},
"user_quota": { "enabled": false,
"max_size_kb": -1,
"max_objects": -1},
"temp_url_keys": []}
```
其中access_key和secret_key需要用来访问S3.

### 测试S3访问

要用S3访问网关，需要提供aws_access_key_id 和aws_secret_access_key和网关的hostname。  
新建s3test.py, 加入：  
```
import boto
import boto.s3.connection
access_key = 'I0PJDPCIYZ665MW88W9R'
secret_key = 'dxaXZ8U90SXydYzyS5ivamEP20hkLSUViiaR+ZDA'
conn = boto.connect_s3(
aws_access_key_id = access_key,
aws_secret_access_key = secret_key,
host = '{hostname}',
is_secure=False,
calling_format = boto.s3.connection.OrdinaryCallingFormat(),
)
bucket = conn.create_bucket('my-new-bucket')
for bucket in conn.get_all_buckets():
        print "{name}\t{created}".format(
                name = bucket.name,
                created = bucket.creation_date,
)
```

运行python 脚本:   
python s3test.py

输出结果：  
```
my-new-bucket ......
```

## 常见问题

### Radosgw 启动失败

#### Initialization timeout
如果出现这种的错误，一种可能的情况就是Ceph集群没有启动，或者集群的状态  
不是HEALTH_OK, 所以启动时候失败。  

还有就是由于配置文件没有在 /etc/ceph/ 目录下，要先确认/etc/ceph/目录下有  
ceph.conf 和keyring文件, 而且都要有可读的权限。  

#### Rados
这种情况，通过查看详细的日志，可以发现是由于认证失败. 我们上面在Ceph的管理  
节点上创建了keyring和用它创建gateway 用户和gateway的key.我们要确认Ceph集群  
真的有加入认证列表中. 用如下命令查看:  
**sudo ceph auth list **

结果应该要有 client.radsogw.gateway。
```

```


### S3测试返回错误

#### 500 Internal Server Error
这个可能的一个原因就是漏掉禁用默认site的操作。
解决的方法就是执行:  
** sudo a2dissite 000-default **

另外一个可能就是mod_proxy_fcgi没有启动，或者没有配置错误。  
没有启动，只要启动就可以了:  
**sudo a2enmod proxy_fcgi**

配置错误就是根据Linux的版本和Apache的配置的不同，修改rgw.conf和网关的配置文件。  

#### 403 Forbbiden Error
可能的原因是S3生成的key中有转义符号'/',然后验证的解析错误，所以验证失败。  
解决的方法就是手动处理或者重新生成key, 手动处理就是将转义符号去掉就可以了。  

#### 405 Method No Found
可能原因的是端口错误，可以先用curl或者浏览器查看一下，  
Apache和mod_proxy_fcgi是不是都是正常，curl {gateway hostname}，  
如果是如下的情况, 说明正常。

```
```
如果不是可能是配置rgw.conf， 端口不是80, 会出现这种情况。

## 参考
