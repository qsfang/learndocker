# Docker容器技术学习总结 #
## 5.Docker网络基础 ##

### 5.1 Docker网络相关技术 ###
Docker现有的网络模型主要是通过使用Network namespace、Linux Bridge、Iptables、veth pair等技术实现的。  

- Network namespace：Network namespace主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等。
- Linux Bridge：功能相当于物理交换机，为连在其上的设备（容器）转发数据帧。如docker0网桥。
- Iptables：主要为容器提供NAT以及容器网络安全。
- veth pair：两个虚拟网卡组成的数据通道。在Docker中，用于连接Docker容器和Linux Bridge。一端在容器中作为eth0网卡，另一端在Linux Bridge中作为网桥的一个端口。

#### 5.1.1 Linux Bridge简介 

**桥接：**桥接就是把一台机器上的若干个网络接口“连接”起来。其结果是，其中一个网口收到的报文会被复制给其他网口并发送出去。以使得网口之间的报文能够互相转发。

**Linux bridge实现：**linux内核支持网口的桥接（目前只支持以太网接口）。但是与单纯的交换机不同，交换机只是一个二层设备，对于接收到的报文，要么转发、要么丢弃。小型 的交换机里面只需要一块交换芯片即可，并不需要CPU。而运行着linux内核的机器本身就是一台主机，有可能就是网络报文的目的地。其收到的报文除了转 发和丢弃，还可能被送到网络协议栈的上层（网络层），从而被自己消化。linux内核是通过一个虚拟的网桥设备来实现桥接的。这个虚拟设备可以绑定若干个以太网接口设备，从而将它们桥接起来。

![](pics\linux-bridge.jpg)  
图5-1 Linux Bridge示意图

如图5-1所示，网桥设备br0绑定了eth0和eth1。对于网络协议栈的上层来说，只看得到br0，因为桥接是在数据链路层实现的，上层不需要关心桥接的细节。于是协 议栈上层需要发送的报文被送到br0，网桥设备的处理代码再来判断报文该被转发到eth0或是eth1，或者两者皆是；反过来，从eth0或从eth1接 收到的报文被提交给网桥br0的处理代码，在这里会判断报文该转发、丢弃、或提交到协议栈上层。而有时候eth0、eth1也可能会作为报文的源地址或目的地址，直接参与报文的发送与接收（从而绕过网桥）。

**网桥功能：**  

- MAC地址学习：建立建立地址-端口的对照表（CAM表）  
- 数据帧转发：每发送一个数据包，网桥都会提取其目的MAC地址，根据地址-端口对照表(CAM表)转发

**Linux bridge配置：**  
1. Linux内核支持网桥：打开Linux内核的CONFIG\_BRIDGE或CONDIG\_BRIDGE\_MODULE编译选项  
2. 安装用户空间工具brctl：通过一系列的ioctl调用来配置网桥  
3. brctl配置示例：

	Brctl addbr br0 # 建立一个网桥br0, 同时在Linux内核里面创建虚拟网卡br0
	Brctl addif br0 eth1 #分别为网桥br0添加接口eth1, eth2和eth3
	Brctl addif br0 eth2
	Brctl addif br0 eth3 

> 其中br0作为一个网桥，同时也是虚拟的网络设备，它即可以用作网桥的管理端口，也可作为网桥所连接局域网的网关。使用br0接口时，必需为它分配IP地址，且与eth1,eth2,eth3在同一网段。

#### 5.1.2 Network NameSpace 与 Veth Pair简介

**Network NameSpace:**  可以实现网络的隔离。  
**veth pair**： 用于不同network namespace间进行通信的方式，veth pair将一个network namespace数据发往另一个　　　　　　　　　　network namespace的veth(如图5-2)。

配置命令（图5-2）：

	# add the namespaces
	ip netns add ns1
	ip netns add ns2
	# create the veth pair
	ip link add tap1 type veth peer name tap2
	# move the interfaces to the namespaces
	ip link set tap1 netns ns1
	ip link set tap2 netns ns2
	# bring up the links
	ip netns exec ns1 ip link set dev tap1 up
	ip netns exec ns2 ip link set dev tap2 up

![](pics\simple-veth-pair.png)  
图5-2 Simple Veth Pair

**如果多个network namespace需要进行通信，则需要借助bridge（图5-3）：**  

配置命令（图5-3）：
	
	# add the namespaces
	ip netns add ns1
	ip netns add ns2
	# create the switch
	BRIDGE=br-test
	brctl addbr $BRIDGE
	brctl stp   $BRIDGE off
	ip link set dev $BRIDGE up
	#

	#### PORT 1
	# create a port pair
	ip link add tap1 type veth peer name br-tap1
	# attach one side to linuxbridge
	brctl addif br-test br-tap1
	# attach the other side to namespace
	ip link set tap1 netns ns1
	# set the ports to up
	ip netns exec ns1 ip link set dev tap1 up
	ip link set dev br-tap1 up

	#
	#### PORT 2
	# create a port pair
	ip link add tap2 type veth peer name br-tap2
	# attach one side to linuxbridge
	brctl addif br-test br-tap2
	# attach the other side to namespace
	ip link set tap2 netns ns2

	# set the ports to up
	ip netns exec ns2 ip link set dev tap2 up
	ip link set dev br-tap2 up
	#

![](pics\veth-pair-linuxbridge.png)  
图5-3 Linux Bridge with two Veth Pairs  
**Veth内核实现：** `//drivers/net/veth.c`

#### 5.1.3 Linux MACVLAN 和 IPVLAN简介 
对应 Docker MACVLAN和 IPVLAN网络驱动。。。
详细见Blog:
[http://blog.csdn.net/dog250/article/details/45788279](http://blog.csdn.net/dog250/article/details/45788279)



### 5.2 Docker容器网络模型和驱动 ###

#### 5.2.1 Libnetwork网络模型 ####
Docker网络成为广为诟病的一大缺陷，在部署大规模Docker集群时，网络也成为了最大的挑战。纯粹的Docker原生网络功能无法满足广大云计算厂商的需要，于是一大批第三方SDN解决方案如雨后春笋般涌现出来，如Pipework、Weave、Flannel、SockertPlane等。2015年3月，Docker宣布收购SocketPlane，随后SocketPlane开始沉寂，一个新的Docker子项目“Libnetwork”开始酝酿。一个月后，libnetwork在Github上正式与开发者见面，预示着Docker开始在网络方面发力。

Libnetwork项目从lincontainer和Docker代码的分离早在Docker 1.7版本就已经完成了（从Docker 1.6版本的网络代码中抽离）。在此之后，容器的网络接口就成为了一个个可替换的插件模块。概括来说，libnetwork提出了新的容器网络模型（Container Network Model，简称CNM），定义了标准的API用于为容器配置网络。只要符合这个模型的网络接口就能被用于容器之间通信，而通信的过程和细节可以完全由网络接口来实现。

![](pics\CNM.jpg)  
图5-5 CNM概念模型

如图5-5所示，CNM网络模型中定义了三个的术语：Sandbox（沙盒）、Endpoint（端点）和Network，它们分别是容器通信中『容器网络环境』、『容器虚拟网卡』和『主机虚拟网卡/网桥』的抽象。

- Sandbox：对应一个容器中的网络运行环境，保存了容器网络栈的配置，包括相应的网卡配置、路由表、DNS配置等。在Linux平台上，沙盒是用Linux Network Namespace实现的。一个沙盒可以包括来自多个网络的多个Endpoint。CNM很形象的将它表示为网络的『沙盒』，因为这样的网络环境是随着容器的创建而创建，又随着容器销毁而不复存在的。
- Endpoint：Endpoint将沙盒加入一个网络，Endpoint的实现可以是一对veth-pair或者OVS内部端口，当前Libnetwork使用的是veth-pair。一个Endpoint只能隶属于一个sandbox及一个network。通过给沙盒增加多个Endpoint可以将一个沙盒加入多个网络。Endpoint实际上就是一个容器中的虚拟网卡，在容器中会显示为eth0、eth1依次类推。
- Network：一个network是一组可以相互通信的endpoints组成。一个network的实现可以是linux bridge，vlan或者其他方式。一个网络中可以包含很多个endpoints。

从CNM的概念角度讲，Libnetwork的出现使得Docker具备了跨主机多子网的能力，同一个子网内的不同容器可以运行在不同的主机上。换言之，位于不同主机上的同一子网（L3）内的容器间可以直接通信，而处于不同子网内的容器即使处于同一主机也不能互通。

Docker内置的驱动包括bridge、host、null、overlay、macvlan和ipvlan，remote是外挂驱动，需要第三方提供网络驱动程序，ipvlan驱动和macvlan驱动是docker1.11新增加的两个测试性模式。
Libnetwork实现了五种驱动（driver）:  

- **null:**容器配置为空，需要手动为容器配置网络接口及路由等   
- **host:**容器与主机共享同一Network Namespace，共享同一套网络协议栈、路由表和iptables规则等。容器与主机看到的是相同的网络视图。 
- **bridge：**Docker设计的NAT网络模型,Docker默认的容器网络驱动。Docker通过一对veth pair连接到 docker网桥上，由Docker为容器动态分配IP及配置路由、防火墙规则等。
> 在bridge驱动方式下，创建sandbox就是在创建linux netnamespace，创建network就是创建linux bridge，创建endpoint就是创建一对veth虚拟设备，通过linux brige和veth虚拟设备来实现不同linux net namespace之间的网络通信。

- **remote：**Docker网络插件的实现。Remote driver使得Libnetwork可以通过HTTP RESTful API对接第三方的网络方案。类似SocketPlane的SDN方案只要实现了约定的HTTP URL处理函数及底层的网络接口配置方法，就可以替换Docker原生的网络实现。remote是外挂驱动，需要第三方提供网络驱动程序。
- **overlay：**Docker原生的跨主机多子网网络方案。主要通过使用Linux bridge和vxlan隧道实现，底层通过类似于**etcd**或**consul**的KV存储系统实现多机的信息同步。
- **ipvlan**和**macvlan：**：驱动docker1.11新增的两个试验性质的驱动包。 

**对应Docker基本网络配置:**

- **none模式：**    　　  命令：`$docker run --net=none`
- **host模式：** 　　     命令：`$docker run --net=host`  
- **container模式：** 容器和另外一个容器共享Network namespace。 kubernetes中的pod就是多个容器共享一个Network namespace。  命令：`$docker run --net=container:<CONTAINER ID>`
- **bridge模式：** : 默认
- **overlay模式（Docker1.9-）：**  命令： `docker network create -d overlay ...`
- **Macvlan模式（Docker1.10-）：** 命令： `docker network create -d macvlan ...`
- **Ipvlan（Docker1.10-）模式**: 命令：`docker network create -d ipvlan ... `

#### 5.2.2 Bridge Drivers：

Docker网络的初始化动作包括：

- 创建docker0网桥
- 为docker0网桥新建子网及路由
- 创建相应的iptables规则

Docker daemon启动时会在主机创建一个Linux网桥（默认是docker0,可通过-b参数手动指定）。容器启动时，Docker会创建一对veth-pair(虚拟网络接口)设备，veth设备的特点是成对存在，从一端进入的数据会同时出现在另一端。Docker会将一端挂载到docker0网桥上，另一端放入容器的Network Namespace内，从而实现容器与主机通信的目的。Bridge模式下Docker容器的网络连接如图5-4所示。容器eth0网卡从docker0网桥所在的IP网段中选取一个未使用的IP，容器的IP在容器重启的时候会改变。docker0的IP为所有容器的默认网关。  

![](pics\docker-bridge.png)  
图5-4 Bridge 模式下的Docker 网络连接图

**容器与外界通信为NAT，容器对外是不可见的。在桥接模式下，Docker容器与Internet的通信，以及不同容器之间的通信都是通过iptables规则控制的。** docker网络相关的命令会转换成对应的iptables规则，例如：  

1. 执行docker端口映射命令：
	`docker run -d --name web -p 80:80 fmzhen/simpleweb`
2. 等价于增加下面一条iptables规则：   
	`-A DOCKER ! -i docker0 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.17.0.5:80`  
此条规则就是对主机eth0收到的目的端口为80的tcp流量进行DNAT转换，将流量发往172.17.0.5:80(容器IP:port)。因此，外界只需访问10.10.101.105:80就可以访问到容器中得服务。



#### 5.2.3 Overlay Network Drivers ####
 Overlay网络模型比较复杂，底层需要类似consul或etcd的KV存储系统进行消息同步，核心是通过Linux网桥与Vxlan隧道实现跨主机划分子网。  
 如图5-5所示，每创建一个网络，Docker会在主机上创建一个单独的沙盒，沙盒的实质是是一个Network Namespace。在沙河中，Docker会创建名为br0的网桥，并在网桥上增加一个Vxlan接口,每个网络占用一个vxlan ID，当前Docker 创建vxlan隧道的ID为256-1000，因而最多可以创建745个网络。当添加一个容器到某一个网络上时，Docker会创建一对veth网卡设备，一段连接到此网络相关沙盒的br0网桥上，另一端放入容器的沙盒内，并设置br0的IP地址作为容器路由默认的网关地址，从而实现加入网络的目的。

![](pics\docker-overlay.jpg)
图5-5 Overlay模式的网络拓扑图

以图5-5为例，容器1和容器4同属于一个网络，容器1需要通过256号VxLAN隧道访问另一台容器4。Docker通过VxLAN和Linux网桥实现可跨主机的虚拟子网功能。

Docker Overlay命令实例：
	##主机1
	$ docker network create -d overlay dev  #创建overlay 网络
	$ docker run -tid --publish-service test.dev ubuntu：latest bash  #启动overlay 网络
	$ docker service ls
	$ docker exec -ti <container ID> ip addr show
	
	##主机2
	$ docker run -tid --publish-service test1.dev.overlay ubuntu：latest bash 
	$ docker service ls

服务test和test1绑定的两个容器即使在两台不同的主机上，相互之间也是可以直接通信的。若另外再创建一个名为prob的网络，并在容器上运行，则无法与dev网络上的容器通信。换言之，网络名与VxLAN ID绑定了？？
		

#### 5.2.4 Macvlan and Ipvlan Network Drivers  
 
详细信息详见：[https://github.com/docker/docker/blob/master/experimental/vlan-networks.md](https://github.com/docker/docker/blob/master/experimental/vlan-networks.md)

对于ipvlan驱动，需要linux kernel版本大于等于4.2才能使用，支持ipvlan的二层和三层两种模式；对于macvlan驱动，需要linux kernel版本大于等于3.9才能使用，支持macvlan的四种模式，分别是private、vepa、bridge和passthru。

在ipvlan驱动方式下，创建sandbox就是在创建linux netnamespace，创建network就是在物理网卡上创建类型是ipvlan的虚拟link网络接口，创建endpoint就是给新创建的虚拟link网络接口设置mac地址和IP地址网络信息，图5-6为ipvlan驱动下sandbox、network和endpoint的实现示意图（endpoint的概念其实可以忽略不计），其中红色字体表示在linux上的具体实现，虚线方块表示libnetwork的逻辑概念：

ipvlan驱动方式不需要通过linux bridge这一层进行处理，所以ipvlan方式效率更高。随着linux kernel中ipvlan驱动的成熟，随着docker对ipvlan支持的完善，将来ipvlan有望成为docker网络通讯中效率最高的一种方式。

**1.MacVlan Bridge Mode使用实例：**  

![](pics\macvlan_bridge_simple.png)  
 图5-6 MacVlan Bridge Mode Example Usage  
Create the macvlan network：

	# Macvlan  (-o macvlan_mode= Defaults to Bridge mode if not specified)
	docker network create -d macvlan \
    	--subnet=172.16.86.0/24 \
    	--gateway=172.16.86.1  \
    	-o parent=eth0 pub_net


**2.IPVlan L2 Mode使用实例：** 

![](pics\ipvlan_l2_simple.png)  
图5-7 Ipvlan L2 Mode Example Usage   


Create the ipvlan network:

	# Ipvlan  (-o ipvlan_mode= Defaults to L2 mode if not specified)
	docker network  create -d ipvlan \
    	--subnet=192.168.1.0/24 \ 
    	--gateway=192.168.1.1 \
    	-o ipvlan_mode=l2 \
    	-o parent=eth0 db_net

**3. IPVlan L3 Mode 使用实例：** 

![](pics\ipvlan-l3.png)  
图5-9 IPVlan L3 Mode Example

**Create the Ipvlan L3 network:**

	docker network  create  -d ipvlan \
    	--subnet=192.168.214.0/24 \
    	--subnet=10.1.214.0/24 \
     	-o ipvlan_mode=l3 ipnet210

