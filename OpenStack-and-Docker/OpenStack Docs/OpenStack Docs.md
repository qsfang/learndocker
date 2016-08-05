# [OpenStack Docs](http://docs.openstack.org/ "OpenStack docs") 学习#
----------
## Release Notes ##
可根据需要查阅Release
### OpenStack Projects Release Notes ###
- 记录OpenStack项目的各个Service Projects的版本release变迁和简介，源码、链接等

### OpenStack Documentation Release Notes ###
- 记录新版本的各种技术支持文档的内容添加和改变等。

## Install Guides ##
介绍如何在各种Linux环境下安装OpenStack的各个组件和服务，支持openSUSE and SUSE Linux Enterprise，Red Hat Enterprise Linux and CentOS，和Ubuntu等操作系统。

### OpenStack Installation Guide for Red Hat Enterprise Linux and CentOS ###

#### 示例架构 硬件需求： #### 
NIC-->Networking Interface Card
![](hwreqs.png)

1. Compute: The compute node runs the hypervisor portion of Compute that operates instances. **By default, Compute uses the KVM hypervisor.**

2. 管理网络，存储网络:For simplicity, service traffic between compute nodes and this node uses the management network. Production environments should implement a separate storage network to increase performance and security.
 
3. Block Storage: 可部署多于一个节点。

4. Object Storage： This service requires two nodes. Each node requires a minimum of one network interface. You can deploy more than two object storage nodes.

#### Network Option: ####
Network option 1 lacks support for self-service (private) networks, layer-3 (routing) services, and advanced services such as LBaaS and FWaaS. Consider the self-service networks option if you desire these features.
![](network1-services.png)
![](network2-services.png)

后面是各种模块安装Blabla...