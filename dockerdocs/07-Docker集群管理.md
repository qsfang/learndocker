# Docker容器技术学习总结 #

## 7.Docker集群管理##

Docker社区中提供三大编排工具：Compose、Machine和Swarm,其功能如下：

- **Docker Machine：**是支持多平台安装Docker的工具，使用Docker Machine可以很方便地在笔记本、云平台及数据中心安装Docker。
- **Docker Swarm：**是Docker社区原生提供的容器集群管理工具。docker集群环境和调度策略等。
- **Docker Compose：** 是用来组装多容器应用的工具，可以在Swarm集群中部署分布式应用。

### 7.1 Docker Machine ###

#### 7.1.1 Docker Machine概述 ####

Docker Machine是一个简化Docker安装的命令行工具，通过一个简单的命令行即可在相应的平台(笔记本、数据中心主机、云主机等)上安装Docker。简单来说，一个Docker Machine是一个Docker host主机和经过配置的Docker client的结合体。  

从技术上讲，Docker Machine就是一个框架，比较开放。对于任何提供虚拟机服务的平台，只要在这个框架下开发针对该平台的驱动，Docker Machine就可以与其交互，可在该平台上执行创建、删除Machine等操作，并且可控制Machine的行为（停止、启动等）。

#### 7.1.2 Docker Machine基本概念及运行流程 ####

Docker Machine会首先创建一个虚拟机并在其上运行一个Docker host，然后使用Docker client和Docker host通信，从而在Docker host上创建镜像、启动容器。

Docker Machine创建虚拟机的时候需要指定相应的驱动，目前支持在本机运行virtualbox虚拟主机，Hyper-V虚拟主机，VMware虚拟主机，AWS EC2，Azure，DigitalOcean，Google等公有云主机，以及使用Openstack搭建的私有数据中心。新的虚拟化（Xen，KVM）支持以及新的云平台支持可以通过开发驱动的方式支持。

每一个Docker Machine创建的虚拟机都要有一个操作系统，默认情况下，VirtualBox驱动使用boot2docker(运行docker容器的轻量级Linux操作系统)。对于云平台的驱动所创建的虚拟机，默认操作系统数ubuntu12.04+。

Docker Machine创建的Docker host的IP地址是所创建的虚拟机的IP地址。

使用Docker Machine及VirtualBox驱动创建本地虚拟机搭建Docker Host的运行流程如下：  
1） 运行`docker-machine create create --driver virtualbox dev`命令。此命令首先创建用于Docker client和Docker host通信的CA证书。其次创建VirtualBox虚拟机，并配置用于通信的TLS参数及网络配置。最后部署Docker的运行环境，即Docker host。  
2）在Docker client里运行`eval "$(docker-machine env dev)"`命令，配置用于和Docker host通信的环境变量。  
3）使用docker相关命令创建或启动相应的容器，如使用`docker run busybox echo hello world`命令可运行busybox工具集。


#### 7.1.3 Docker Machine实战 ####
**1. Docker Machine本机安装**

	$ docker-machine -v
	$  docker-machine ls
 	#已安装了 VirtualBox，并且要创建一个叫“testing”的虚拟机： 
	$  docker-machine create --driver virtualbox testing
	#docker-machine docker client连接 docker host 
	$  docker-machine env testing
	$  docker-machine config testing
	--tls --tlscacert=/Users/russ/.docker/machine/machines/testing/ca.pem --tlscert=/Users/
	russ/.docker/machine/machines/testing/cert.pem --tlskey=/Users/russ/.docker/machine/machines/
	testing/key.pem -H="tcp://192.168.99.100:2376

	#启用一个虚拟机并准备使用Docker。 
	$  docker-machine ls
	#运行一个“Hello World”： 
	$  docker $(docker-machine config testing) run busybox echo hello world

	#使用docker-machie ssh machine-name命令SSH到虚拟机： 
	$  docker-machine ssh testing

**2. docker-machine启用Digital Ocean的实例**

	docker-machine访问Digital Ocean账户通过API并且启用实例：
	$  docker-machine create --driver digitalocean --digitalocean-access-token <Token> 	dotesting
 
	#以实例dotesting运行“Hello World”:
	$  docker $(docker-machine config dotesting) run busybox echo hello world

### 7.2 Docker Swarm  ###

### 7.3 Docker Compose ###