# Docker容器技术学习总结 #
## 4.Docker Image 扩展知识--Docker文件系统 ##
Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。最初Docker仅能在支持Aufs文件系统的Linux发行版上运行，但是由于Aufs未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。目前，Docker支持**Aufs，Devicemapper**，Btrfs和Vfs四种文件系统。

> **OverlayFS**  《Docker进阶与实践》3.4节  
> 联合挂载，写时复制，覆盖，新增，删除

本文详述：**Aufs与Devicemapper**

### 4.1 Linux文件系统 ###
典型的Linux文件系统由bootfs和rootfs两部分组成，bootfs(boot file system)主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后， bootfs就被umount了。 rootfs (root file system) 包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。
![](pics\linuxfs.png)  

图4-1 Linux文件系统

#### 4.1.1 Aufs文件系统 ####
Aufs是一种Union FS， 简单来说就是支持将不同的目录挂载到同一个虚拟文件系统下，并实现一种layer的概念。Aufs将挂载到同一虚拟文件系统下的多个目录分别设置成read-only，read-write以及whiteout-able权限，对read-only目录只能读，而写操作只能实施在read-write目录中。重点在于，写操作是在read-only上的一种增量操作，不影响read-only目录。当挂载目录的时候要严格按照各目录之间的这种增量关系，将被增量操作的目录优先于在它基础上增量操作的目录挂载，待所有目录挂载结束了，继续挂载一个read-write目录，如此便形成了一种层次结构。

在挂载的过程中，mount命令按照命令行中给出的文件夹顺序挂载，若出现有同名文件的情况，则以先挂载的为主，其他的不再挂载。Docker镜像为什么采用增量的方式：完全是利用Aufs的特性达到节约空间的目的。

#### 4.1.2 Devicemapper文件系统####

**1.Linux的Logical Volume：Thinly-Provisioned Snapshot原理**  


Snapshot是Lvm提供的一种特性，它可以在不中断服务运行的情况下为the origin（original device）创建一个虚拟快照(Snapshot)，它具有以下几个特点：

1. 当the origin内容发生变化时，snapshot对变化的部分做一个拷贝以用来对the origin进行重构。
2. 因为只对变化的部分做拷贝，所以Lvm的Snapshot在读操作频繁而写操作不频繁的情况下占用很少的一部分空间便能完成特定任务。
3. 当Snapshot大小耗尽或者远大于实际需求时，我们可以对其大小进行调节。
4. 当对Snapshot的数据进行写操作的时候，Snapshot实施相应操作，并丢弃从the origin的拷贝，以后的操作以写操作之后Snapshot中的数据为准。  
5. 在某些发行版的Linux系统下，可以使用lvconvert的--merge选项将Snapshot合并回the origin。

Thin-Provisioning是一项利用虚拟化方法减少物理存储部署的技术，可最大限度提升存储空间利用率。下图中展示了某位用户向服务器管理员请求分配10TB的资源的情形。实际情况中这个数值往往是峰值，根据使用情况，分配2TB就已足够。因此，系统管理员准备2TB的物理存储，并给服务器分配10TB的虚拟卷。服务器即可基于仅占虚拟卷容量1/5的现有物理磁盘池开始运行。这样的“始于小”方案能够实现更高效地利用存储容量。

![](pics\Thin-Provisioning.png)  
图4-2 Thin-Provisioning


Thin-provisioning Snapshot结合Thin-Provisioning和Snapshot两种技术，允许多个虚拟设备同时挂载到一个数据卷以达到数据共享的目的。Thin-Provisioning Snapshot的特点如下：

1. 可以将不同的snaptshot挂载到同一个the origin上，节省了磁盘空间。
2. 当多个Snapshot挂载到了同一个the origin上，并在the origin上发生写操作时，将会触发COW操作。这样不会降低效率。
3. Thin-Provisioning Snapshot支持递归操作，即一个Snapshot可以作为另一个Snapshot的the origin，且没有深度限制。
4. 在Snapshot上可以创建一个逻辑卷，这个逻辑卷在实际写操作（COW，Snapshot写操作）发生之前是不占用磁盘空间的。

Thin-Provisioning Snapshot虽然有诸多优点，但是也有很多不足之处，例如大小固定等问题。T

**2.DeviceMapper:Thinly-Provisioned Snapshot实现**  
Thin-Provisioning Snapshot是作为device mapper的一个target在内核中实现的。Device mapper 是Linux 2.6内核中提供的一种从逻辑设备到物理设备的映射框架机制。在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略，如条带化，镜像，快照等。

Device Mapper主要包含内核空间的映射和用户空间的device mapper库及dmsetup工具。Device Mapper库是对ioctl、用户空间创建删除Device Mapper逻辑设备所需必要操作的封装，dmsetup是一个提供给用户直接可用的创建删除device mapper设备的命令行工具。

我们以dmsetup命令来介绍一下Thin-Provisioning Snapshot时如何实现的。Thin-Provisioning Snapshot需要一个data设备和一个metadata设备分别用来存放实际数据和元数据。有两种方式可以更改metadta：一种时通过函数调用，另一种则是通过dmsetup message命令。这里，我们创建两个稀疏文件作为data和metadata设备：

	[root@qingze dev]# dd if=/dev/zero of=/tmp/metadata bs=1K count=1 seek=2G
	1+0 records in
	1+0 records out
	1024 bytes (1.0 kB) copied, 0.000153611 s, 6.7 MB/s
	[root@qingze dev]# dd if=/dev/zero of=/tmp/data bs=1K count=1 seek=100G
	1+0 records in
	1+0 records out
	1024 bytes (1.0 kB) copied, 8.7567e-05 s, 11.7 MB/s

以上命令分别创建了2G的metadata和100G的data两个稀疏文件。注意命令中seek选项，其表示为略过of选项指定的输出文件的前2G空间再写入内容。这2G在硬盘上是没有占有空间的，占有空间只有1k的内容。当向其写入内容时，才会在硬盘上为其分配空间。

文件创建完成后，我们将其挂载到loopback设备上，以供使用：

	[root@qingze dev]# losetup /dev/loop0 /tmp/metadata
	[root@qingze dev]# losetup /dev/loop1 /tmp/data
	[root@qingze dev]# losetup -a
	/dev/loop0: [0035]:183033 (/tmp/metadata)
	/dev/loop1: [0035]:185651 (/tmp/data)

准备工作就绪后，我们开始正式创建Snapshot。首先创建一个thin-pool，指定要用哪些设备来创建Snapshot，此命令的作用为：将逻辑设备的0～20971520之间的sector映射到/dev/loop1上，元数据信息存储到/dev/loop1：

	[root@qingze dev]# dmsetup create pool --table "0 20971520 thin-pool 
	/dev/loop0 /dev/loop1 128 32768 1 skip_block_zeroing"
- pool:指定thin-pool的名字
- --table指定dmsetup的一个target，其中：0表示起始sector 20971520表示sector的数量
- thin-pool表示target的名字
- /dev/loop0 /dev/loop1为设备名
- 128表示一次可分配的最小sector数
- 32768表示空闲空间的阈值
- 1特征参数的个数
- skip_block_zeroing特征参数，表示略过用0填充的块

在继续创建Snapshot前，需要创建一个Thinly-Provisioned volume并格式化：

	[root@qingze dev]# dmsetup message /dev/mapper/pool 0 "create_thin 0"
	[root@qingze dev]# dmsetup create thin --table "0 2097152 thin /dev/mapper/pool 0"
	[root@qingze dev]# mkfs.ext4 -E discard,lazy_itable_init=0 /dev/mapper/thin
命令的格式为：

	dmsetup message device_name sector message
	Send message to target. If sector not needed use 0
	'0' is an identifier for the volume

官方文档中对Snapshot有internal和external之分，说明如下：  

internal snapshot:  
Once created, the user doesn't have to worry about any connection
between the origin and the snapshot. Indeed the snapshot is no
different from any other thinly-provisioned device and can be
snapshotted itself via the same method. It's perfectly legal to
have only one of them active, and there's no ordering requirement on
activating or removing them both. (This differs from conventional
device-mapper snapshots.)

external snapshot:  
You can use an external read only device as an origin for a
thinly-provisioned volume. Any read to an unprovisioned area of the
thin device will be passed through to the origin. Writes trigger
the allocation of new blocks as usual.

读者可以根据自己的需求选择，这里我们只建立一个internal snapshot：

	[root@qingze dev]# dmsetup suspend /dev/mapper/thin
	[root@qingze dev]# dmsetup message /dev/mapper/pool 0 "create_snap 1 0"
	[root@qingze dev]# dmsetup resume /dev/mapper/thin
	[root@qingze dev]# dmsetup create snap --table "0 2097152 thin /dev/mapper/pool 1"
	[root@qingze dev]# dmsetup status
	thin: 0 2097152 thin 99840 2097151
	fedora-swap: 0 8126464 linear 
	fedora-root: 0 104857600 linear 
	snap: 0 2097152 thin 99840 2097151
	pool: 0 20971520 thin-pool 0 280/4161600 780/163840 - rw discard_passdown queue_if_no_space 
	fedora-home: 0 97509376 linear
至此，我们已经成功建立了一个Snapshot，接下来可以将其挂载到任何一个文件夹下，并对其进行创建文件等操作：

	[root@qingze dev] mount /dev/mapper/snap dv
我们在dv文件夹下创建一个内容为this is snap的文件file，然后，在snap的基础上再创建一个snap1，观察发生了什么：

	[root@qingze tmp]# dmsetup message /dev/mapper/pool 0 "create_snap 2 1"
	[root@qingze tmp]# dmsetup create snap1 --table "0 2097152 thin /dev/mapper/pool 2"
	[root@qingze tmp]# mkdir dv1
	[root@qingze tmp]# mount /dev/mapper/snap1 dv1

	[root@qingze dv]# cd ..
	[root@qingze tmp]# cd dv1
	[root@qingze dv1]# ls
	file  lost+found
	[root@qingze dv1]# cat file
	this is snap

可以发现包含同样的文件和内容。事实上，Docker正是以这种方式来创建容器的，读者可以执行以下命令来验证：

在`metadata`文件夹下查看某个某个容器的`device_id`:

	[root@qingze metadata]# cat 3b363fd9d7dab4db9591058a3f43e806f6fa6f7e2744b63b2df4b84eadb0685a 
	{"device_id":910,"size":10737418240,"transaction_id":1817,"initialized":false}
创建并挂载snapshot:

	[root@qingze tmp]# dmsetup message /dev/mapper/docker-253:1-3020991-pool 0 "create_snap 6001 910"
	[root@qingze tmp]# dmsetup create snap6 --table "0 20971520 thin /dev/mapper/docker-253:1-3020991-pool 6001"
	[root@qingze tmp]# mkdir dv6
	[root@qingze tmp]# mount /dev/mapper/snap6 dv6
	[root@qingze tmp]# cd dv6/
	[root@qingze dv6]# ls
	id  lost+found  rootfs
	[root@qingze dv6]# cd rootfs/
	[root@qingze rootfs]# ls
	bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin 

 
### 4.2 Docker 镜像文件系统 ###

Docker镜像的典型结构如下图4-3。传统的Linux加载bootfs时会先将rootfs设为read-only，然后在系统自检之后将rootfs从read-only改为read-write，然后我们就可以在rootfs上进行写和读的操作了。但Docker的镜像却不是这样，它在bootfs自检完毕之后并不会把rootfs的read-only改为read-write。而是利用union mount（UnionFS的一种挂载机制）将一个或多个read-only的rootfs加载到之前的read-only的rootfs层之上。在加载了这么多层的rootfs之后，仍然让它看起来只像是一个文件系统，在Docker的体系里把union mount的这些read-only的rootfs叫做Docker的镜像。但是，此时的每一层rootfs都是read-only的，我们此时还不能对其进行操作。当我们创建一个容器，也就是将Docker镜像进行实例化，系统会在一层或是多层read-only的rootfs之上分配一层空的read-write的rootfs。
![](pics\docker-img-struc.png)

图4-3 Docker镜像的典型结构

像，使用`docker images -tree`查看结果如下：

	[root@qingze qingze]# docker images -tree
	Warning: '-tree' is deprecated, it will be removed soon. See usage.
		└─511136ea3c5a Virtual Size: 0 B
  			└─3b363fd9d7da Virtual Size: 192.5 MB
   				└─607c5d1cca71 Virtual Size: 192.7 MB
     				└─f62feddc05dc Virtual Size: 192.7 MB
       					└─8eaa4ff06b53 Virtual Size: 192.7 MB Tags: ubuntu:14.04,

可以看到Ubuntu的镜像中有多个长ID的layer，且以一种树状结构继承下来，如下图。其中，第n+1层继承了第n层，并在此基础上有了自己的内容，直观上的表现就是第n+1层占用磁盘空间增大。并且，不同的镜像可能会有相同的父镜像。

接下来，使用docker save命令将镜像具体化并查看其结构：

	[root@qingze qingze]# docker save -o ubuntu.tar ubuntu:14.04
	[root@qingze qingze]#  tar -tf ubuntu.tar 
	3b363fd9d7dab4db9591058a3f43e806f6fa6f7e2744b63b2df4b84eadb0685a/
	3b363fd9d7dab4db9591058a3f43e806f6fa6f7e2744b63b2df4b84eadb0685a/VERSION
	3b363fd9d7dab4db9591058a3f43e806f6fa6f7e2744b63b2df4b84eadb0685a/json
	3b363fd9d7dab4db9591058a3f43e806f6fa6f7e2744b63b2df4b84eadb0685a/layer.tar
	...
	...
	f62feddc05dc67da9b725361f97d7ae72a32e355ce1585f9a60d090289120f73/
	f62feddc05dc67da9b725361f97d7ae72a32e355ce1585f9a60d090289120f73/VERSION
	f62feddc05dc67da9b725361f97d7ae72a32e355ce1585f9a60d090289120f73/json
	f62feddc05dc67da9b725361f97d7ae72a32e355ce1585f9a60d090289120f73/layer.tar
	repositories
我们可以发现在具体化的镜像中对应树型结构中的每一个layer都有一个文件夹存在，且每个文件夹中包含VERSION，json，layer.tar三个文件,另外还有一个repositories文件。长ID为 511136ea3c5a 的layer没有继承任何layer，因此被称为base镜像，一般来说镜像都是以这个layer开始的， 其内容也为空。每个文件夹下都有一个layer.tar的文件，这个压缩文件存放的正是rootfs的内容，不过是增量存放的而已。


### 4.3 Docker创建启动镜像的源码分析 ###  
[http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper](http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper)

### 4.4 总结 ###
- Aufs实现起来比较简单，但是由于其迟迟不能加入linux内核，导致兼容性差。目前，仅有Ubuntu支持。
- Devicemapper虽然实现起来复杂，但兼容性好。其存在的一点不足是，当metadata和data空间被耗尽时，需要重启Docker来扩充空间。