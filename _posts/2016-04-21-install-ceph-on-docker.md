---
layout: post
title: install ceph on docker
tags: docker ceph

---
 

# INTALL CEPH ON DOCKER

我的环境是：
ubuntu LTS 14.04

### 安装DOCKER
安装docker在ubuntu下，可以直接使用apt-get，但是默认安装的不是最新版。
安装docker最新版的如下：
> curl -sSL https://get.daocloud.io/docker | sh

由于在国内网络原因，如果直接从docker Hub下载镜像非常慢，而且容易中间出错。
所以用国内的docker加速器，我用的是daocloud。daocloud注册一个账号，然后
再用docker安装daocloud的加速器，这样每次下载镜像的时候，直接使用dao pull
就可以了，速度快很多。

安装docker的加速器：
> curl -sSL https://get.daocloud.io/daomonit/install.sh | sh -s dec4a3298f2e8da43169f09f18cd5c26ee04b165 


### 安装MONITOR
安装非常简单，只需要使用
dao pull ceph/mon 
利用 docker images 查看镜像是否下载好了。
sudo docker images
[](install-ceph/images.jpg)

构造monitor容器并运行
``` 
sudo docker run -tid --name=mon --net=host
-e MON_NAME=mymon -e MON_IP=192.168.0.136
-e CEPH_PUBLIC_NETWORK=192.168.0.0/24  
-v /etc/ceph:/etc/ceph ceph/mon
```
sudo docker run -tid --name=mon --net=host -e MON_NAME=mon3 -e MON_IP=192.168.56.3 -e CEPH_PUBLIC_NETWORK=192.168.56.0/24 -v /etc/ceph:/etc/ceph ceph/mon

--name 为在docker镜像cpeh/mon设置一个名字，方便使用
--net  ceph/mon 设置网络，为host, none, bridge等
-e 设置环境变量相关的参数
-v 设置挂载卷

查看当前正在运行的容器
> sudo docker ps
[](install-ceph/ps.jpg)

### 修改配置文件
配置文件在/etc/ceph/ceph.conf
添加
``` 
osd pool default size = 1  这个设置的是备份数
osd pool default mini size = x 如何上一行为1，下面就可以不用设置了。
osd pool default pg num = 32  这个是设置每个资源池pool的放置组数目
osd pool default pgg num = 32
osd crush chooseleaf num = 0 设置的是集群节点（单集群设置为0）
``` 
注意：
如果是单集群，osd crush chooseleaf num 要设置为0
osd pool default size 为备份数，跟集群的存储节点有关，要求存储节点数目
不小于备份数。
存储节点（storage node）: 指的是集群中osd所在的host数目，也就是说与osd
的数量没有关系，而是与osd的host有关系。如果多个osd都是在同一个host下，
那么是属于同一个storage node的。可以用 docker exec mon ceph osd tree
查看。

### 安装OSD
和MONITOR类似，使用
dao pull ceph/osd

在构建osd前需要现在集群上创建osd，这样monitor才能将osd加入到集群中。
sudo docker exec mon ceph osd create

构造osd容器并运行
``` 
sudo docker run -tid --name=osd0 --net=host
-e CLUSTER=ceph -e WEIGHT=1.0
-e MON_NAME=mymon -e MON_IP=192.168.0.136
-v /etc/ceph:/etc/ceph
-v /opt/osd/0:/var/lib/ceph/osd/ceph-0 ceph/osd
```

sudo docker run -tid --name=osd0 --net=host -e CLUSTER=ceph -e WEIGHT=1.0 -e MON_NAME=mon1 -e MON_IP=192.168.56.1 -v /etc/ceph:/etc/ceph -v /opt/osd/0:/var/lib/ceph/osd/ceph-0 ceph/osd
这里我们osd容器的net为host，就是为localhost，为一个存储节点。
-e 设置的是osd的环境变量先关的。监控的IP等信息。
-v /opt/sod/0 是osd挂载的地方。
如果需要增加osd容器，则需要修改上面的参数。将0改为其他数字。

### 查看Ceph集群的日志
使用docker查看容器的日志  
**sudo docker logs -f container**      
如查看ceph的日志：   
**sudo docker logs -f mon** 

[](install-ceph/logs.jpg)

查看Ceph集群的状态  
> sudo docker exec mon ceph -s
[](install-ceph/status.jpg)

查看时集群的要是HEALTH_OK, 才可以。
由于集群刚开始创建是会默认创建一个rbd pool，而这个pool是默认为三个副本。
所以如果集群中没有三个存储节点，那么集群的状态就p一直是HEALTH_WARN.
可以用docker exec mon ceph osd dump查看osd中每个pool的情况。

上述情况的解决方法又两种，一种就是构造多两个存储节点(不是指osd，host要不同).
或者直接将这个rbd pool 删除掉，然后再创建新的pool。由于我们上的配置文件，
已经修改为每个pool默认为一个副本，所以就正常了。

创建或者删除pool
``` 
sudo docker exec mon ceph osd create pool_name pg_num
sudo docker exec mon ceph osd delete pool_namm pool_namm --i-really-want-to-do-it
``` 
