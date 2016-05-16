---
layout: post
title: ceph intro
tags: ceph

---


# Ceph介绍

## Ceph简介

Ceph是一个分布式对象存储和文件系统，且致力于提供高性能，可用性和可扩展性，
Ceph最初来源于Sege Weil在University of California, Santa Cruz (UCSC)关于
存储系统的PhD研究项目，现在Ceph是开源的，由其全球性社区的存储工程师和研究者
共同维护。Ceph是统一存储系统，基于RADOS(Reliable, Autonomous, Distributed,
Object, Store)提供对象(Object)，块(Block)，文件(File)三种存储方式。

Ceph存储系统具有如下特点:

* 高可扩展性
  使用普通的X86机器，支持TB到PB级别的数据扩展。
* 高可靠性
  没有单点故障，多数据备份，自动管理，自动修复。
* 高性能
  数据均衡分布，并行化程度高。对象存储和块存储时，不需要元数据服务器。

## Ceph架构

Ceph生态系统大致可划分为四个部分:

* 客户端  用户使用Ceph入口
* 元数据服务器  缓存或同步元数据
* 对象存储集群  将数据和元数据作为对象存储
* 集群监控器  执行监视功能

![](./pictures/ceph-arch.png)

Figure 1. Ceph生态系统概念架构

客户端使用元数据服务器执行元数据操作来确定数据位置，元数据服务器管理元数据
的位置以及在何处存储新数据。实际上，元数据也是存储在对象存储集群上的，而数
据文件的实际I/O则是发生在客户端和对象存储集群间。如此一来，更高层次的POSIX
功能(如，打开，关闭，重命名)就由元数据服务器管理，而POSIX功能(读和写)则由对
像存储集群管理。

## Radosgw

Ceph存储系统支持对象存储，对象存储通过Rados网关进行访问Ceph存储集群。Ceph对
象网关是建立在Librados之上的对象存储接口，为应用提供Ceph存储集群的Restful
网关。 Ceph对象存储支持两种兼容接口： 兼容Amazon S3接口和兼容Swift接口。Ceph
对象存储使用Radosgw与Ceph存储集群交互，Radosgw是一个FastCGI模块。由于Radosgw
提供了兼容Openstack Swift接口和Amazon S3接口，对象存储网关有自己的用户管理。

![](./picture/radosgw.png)

Figure 2. Rados Gateway

## 参考资料

[Ceph](http://ceph.com/)

[Ceph Object GateWay](http://docs.ceph.com/docs/master/radosgw/)


