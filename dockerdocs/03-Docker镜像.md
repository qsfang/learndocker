# Docker容器技术学习总结 #
## 3.Docker镜像 ##

**Docker 所宣称的用户可以随心所欲地“ Build、Ship and Run”应用的能力，其核心是由Docker image（Docker 镜像）来支撑的。**Docker 通过把应用的运行时环境和应用打包在一起，解决了部署环境依赖的问题；通过引入分层文件系统这种概念，解决了空间利用的问题。它彻底消除了编译、打包与部署、运维之间的鸿沟，与现在互联网企业推崇的DevOps 理念不谋而合，大大提高了应用开发部署的效率。Docker 公司的理念被越来越多的人理解和认可也就是理所当然的了，而理解Docker image 则是深入理解Docker 技术的一个关键点。
### 3.1 Docker image是什么？ ###
**Docker image 是用来启动容器的只读模板，是容器启动所需要的rootfs，类似于虚拟机所使用的镜像。**  

![](pics/docker-img.jpg)  
图3-1　Docker 镜像的典型表示法


图3-1 是典型的Docker 镜像的表示方法，可以看到其被“ /”分为了三个部分，其中每部分都可以类比Github 中的概念。下面按照从左到右的顺序介绍这几个部分以及相关的一些重要概念。

- Remote docker hub ：集中存储镜像的 Web 服务器地址。该部分的存在使得可以区分从不 同镜像库中拉取的镜像。若Docker 的镜像表示中缺少该部分，说明使用的是默认镜像库，即Docker 官方镜像库。
- Namespace：类似于 Github 中的命名空间，是一个用户或组织中所有镜像的集合。
- Repository：类似于 Git 仓库，一个仓库可以有多个镜像，不同镜像通过 tag 来区分。
- Tag：类似 Git 仓库中的 tag，一般用来区分同一类镜像的不同版本。
- Layer：镜像由一系列层组成，每层都用 64 位的十六进制数表示，非常类似于 Git 仓库中的commit。
- Image ID ：镜像最上层的 layer ID 就是该镜像的 ID，Repo:tag 提供了易于人类识别的名字，而ID 便于脚本处理、操作镜像。

**镜像库**是Docker 公司最先提出的概念一，非常类似应用市场的概念。用户可以发布自己的镜像，也可以使用别人的镜像。Docker 开源了镜像存储部分的源代码（Docker Registry以及Distribution），但是这些开源组件并不适合独立地发挥功能，需要使用Nginx 等代理工具添加基本的鉴权功能，才能搭建出私有镜像仓库。

**本地镜像**则是已经下载到本地的镜像，可以使用docker images 等命令进行管理。这些镜像默认存储在/var/lib/docker 路径下，该路径也可以使用docker daemon –g 参数在启动Daemon 时指定。


### 3.2 Docker image的"build,ship,run" ###
Docker 内嵌了一系列命令制作、管理、上传和下载镜像。可以调用REST API 给Docker daemon 发送相关命令，也可以使用client 端提供的CLI 命令完成操作。图3-2总结了Docker image生命周期图。

![](pics/docker-life.jpg) 
图3-2　Docker image 生命周期

 
下面从Docker image 的生命周期角度说明Docker image 的相关使用方法。

**1.列出本机镜像**  

	$ docker images --help
	Usage: docker images [OPTIONS] [REPOSITORY]
	List images
	-a, --all=false Show all images (default hides intermediate images)
	--digests=false Show digests
	-f, --filter=[] Filter output based on conditions provided
	--help=false Print usage
	--no-trunc=false Don't truncate output
	-q, --quiet=false Only show numeric IDs

--no-trunc 参数可以列出完整长度的Image ID。若添加参数-q 则会只输出Image ID，该参数在管道命令中很有用处。

其中，--filter 用于过滤docker images 的结果，过滤器采用key=value 的这种形式。目前支持的过滤器为dangling 和label。 

	$docker images --filter "dangling=true"
>--filter "dangling=true" 会显示所有“悬挂”镜像。“悬挂”镜像没有对应的名称和tag，并且其最上层不会被任何镜像所依赖。docker commit 在一些情况下会产生这种“悬挂”镜像。
一般来说悬挂镜像并不总是我们所需要的，并且会浪费磁盘空间。可以使用如下管道命令删除所有的“悬挂”镜像。

	$ docker images --filter "dangling=true" -q | xargs docker rmi

官方推荐使用[dockerviz](https://github.com/justone/dockviz)工具分析Docker image，可以图形化地展示Docker image的层次关系。dockerviz命令如如下：
	
	# dockviz images -d | dot -Tpng -o images.png

如下命令可显示镜像的所有层(layer)  
	
	$ sudo docker images --tree
	
**2.Build:创建一个镜像**

创建镜像是一个很常用的功能，既可以从无到有地创建镜像，也可以以现有的镜像为基础进行增量开发，还可以把容器保存为镜像。
  
- 直接下载镜像： `docker pull busybox` 
- 导入镜像： `$  docker save [OPTIONS] IMAGE [IMAGE...]`   
　　　　　 `$ docker load -i busybox.tar`
- 制作新镜像： `$ docker import`    
 　　　　　　  `$ docker export [OPTIONS] CONTAINER`
- 将运行的容器保存为镜像：`docker commit <container-id> <image-name>`
> docker load 一般只用于导入由docker save 导出的镜像，导入后的镜像跟原镜像完
全一样，包括拥有相同的镜像ID 和分层等内。  
docker import 用于导入包含根文件系统的归档，并将之变成Docker 镜像。 docker import 常用来制作Docker 基础镜像，如Ubuntu 等镜像；docker export 则是把一个镜像导出为根文件系统的归档。

>**区别：**Export命令用于持久化容器（不是镜像），Save命令用于持久化镜像（不是容器）。导出后再导入(exported-imported)的镜像会丢失所有的历史，而保存后再加载（saved-loaded）的镜像没有丢失历史和层(layer)。这意味着使用导出后再导入的方式，你将无法回滚到之前的层(layer)，同时，使用保存后再加载的方式持久化整个镜像，就可以做到层回滚（可以执行docker tag <LAYER ID\> <IMAGE NAME\>来回滚之前的层）。


**3.Ship：传输一个镜像**

镜像传输是连接开发和部署的桥梁。

- 使用Docker 镜像仓库做中转传输
- 使用docker export/docker save 生成的tar 包来实现
- 使用Docker 镜像的模板文件Dockerfile 做间接传输
> 目前托管在Github 等网站上的项目，已经越来越多地包含有Dockerfile 文件；同时Docker 官方镜像仓库使用了github.com 的webhook 功能，若代码被
修改就会触发流程自动重新制作镜像。

**4.Run:以image 为模板启动一个容器**

命令：`$ docker run`

 
### 3.3 Docker image的组织结构 

Docker image 是用来启动容器的只读模板，提供容器启动所需要的rootfs，那么Docker 是怎么组织这些数据的呢？  
#### 3.3.1 Docker image的数据内容   

Docker image 包含着**数据**及必要的**元数据**。数据由一层层的image layer 组成，元数据则是一些JSON 文件，用来描述数据（image layer）之间的关系以及容器的一些配置信息。下面使用overlay 存储驱动对Docker image 的组织结构进行分析，首先需要启动Dockerdaemon，命令如下：

	# docker daemon -D –s overlay –g /var/lib/docker

首先查看本地存储路径/var/lib/docker：

	# ls -l /var/lib/docker
	total 44
	drwx------ 2 root root 4096 Jul 24 18:41 containers # 存放容器运行相关信息
	drwx------ 3 root root 4096 Apr 13 14:32 execdriver
	drwx------ 6 root root 4096 Jul 24 18:43 graph #Image 各层的元数据
	drwx------ 2 root root 4096 Jul 24 18:41 init
	-rw-r--r-- 1 root root 5120 Jul 24 18:41 linkgraph.db
	drwxr-xr-x 5 root root 4096 Jul 24 18:43 overlay #Image 各层数据
	-rw------- 1 root root 106 Jul 24 18:43 repositories-overlay #Image 总体信息
	drwx------ 2 root root 4096 Jul 24 18:43 tmp
	drwx------ 2 root root 4096 Jul 24 19:09 trust # 验证相 关信息
	drwx------ 2 root root 4096 Jul 24 18:41 volumes # 数据卷相关信息

**1.总体信息**  
从repositories-overlay 文件可以看到该存储目录下的所有image 以及其对应的layerID。为了减少干扰，实验环境之中只包含一个镜像，其ID 为8c2e06607696bd4af，如下。

	# cat repositories-overlay |python -m json.tool
	{
		"Repositories": {
			"busybox": {
					"latest": "8c2e06607696bd4afb3d03b687e361cc43cf8ec1a4a725bc96e39f05ba97dd55"
					}
				}
	}

**2.数据和元数据**  
graph 目录和overlay 目录包含本地镜像库中的所有元数据和数据信息。对于不同的存储驱动，数据的存储位置和存储结构是不同的。可以通过下面的命令观察数据和元数据中的具体内容。元数据包含json 和layersize 两个文件，其中json 文件包含了必要的层次和配置信息，layersize 文件则包含了该层的大小。

	# ls -l graph/8c2e06607696bd4afb3d03b687e361cc43cf8ec1a4a725bc96e39f05ba97dd55/
	total 8
	-rw------- 1 root root 1446 Jul 24 18:43 json
	-rw------- 1 root root 1 Jul 24 18:43 layersize
	# ls -l overlay/8c2e06607696bd4afb3d03b687e361cc43cf8ec1a4a725bc96e39f05ba97dd55/
	total 4
	drwxr-xr-x 17 root root 4096 Jul 24 18:43 root

**3.Docker image还原**  
Docker 镜像存储路径下已经存储了足够的信息，Docker daemon 可以通过这些信息还原出Docker image ：  
　　1. 先通过repositories-overlay 获得image 对应的layer ID ；  
　　2. 再根据layer 对应的元数据梳理出image 包含的所有层，以及层与层之间的关系；  
　　3. 然后使用**联合挂载技术**还原出容器启动所需要的 rootfs 和一些基本的配置信息。  

#### 3.3.2 Docker image的数据组织  

通过repositories-overlay 可以找到某个镜像的最上层layer ID，进而找到对应的元数据，那么元数据都存了哪些信息呢？可以通过docker inspect 得到该层的元数据。为了简单起见，下面的命令输出中删除了一些与讨论无关的层次信息。

	$ docker inspect busybox:latest
	[
	{
		"Id": "8c2e06607696bd4afb3d03b687e361cc43cf8ec1a4a725bc96e39f05ba97dd55",
		"Parent": "6ce2e90b0bc7224de3db1f0d646fe8e2c4dd37f1793928287f6074bc451a57ea",
		"Comment": "",
		"Created": "2015-04-17T22:01:13.062208605Z",
		"Container": "811003e0012ef6e6db039bcef852098d45cf9f84e995efb93a176a11e9aca6b9",
		"ContainerConfig": {
				"Hostname": "19bbb9ebab4d",
				"Domainname": "",
				...
				...
				"Env": null,
				"Cmd": [
					"/bin/sh"
					],
				"Architecture": "amd64",
				"Os": "linux",
				"Size": 0,
				"VirtualSize": 2433303,
				"GraphDriver": {
					"Name": "aufs",
					"Data": null
				}
	}
	]

对于上面的输出，有几项需要重点说明一下：

- Id：Image 的 ID。通过上面的讨论，可以看到 image ID 实际上只是最上层的 layerID，所以docker inspect 也适用于任意一层layer。
- Parent：该 layer 的父层，可以递归地获得某个 image 的所有 layer 信息。
- Comment：非常类似于 Git 的 commit message，可以为该层做一些历史记录，方便其他人理解。
- Container：这个条目比较有意思，其中包含哲学的味道。比如前面提到容器的启动需要以image 为模板。但又可以把该容器保存为镜像，所以一般来说image 的每个layer 都保存自一个容器，所以该容器可以说是image layer 的“模板”。
- Config：包含了该 image 的一些配置信息，其中比较重要的是：“ env”容器启动时会作为容器的环境变量；“ Cmd”作为容器启动时的默认命令；“ Labels”参数可以用于docker images 命令过滤。
- Architecture：该 image 对应的 CPU 体系结构。现在 Docker 官方支持 amd64，对其他体系架构的支持也在进行中。

通过这些元数据信息，可以得到某个image 包含的所有layer，进而组合出容器的rootfs，再加上元数据中的配置信息（环境变量、启动参数、体系架构等）作为容器启动时的参数。至此已经具备启动容器必需的所有信息。

 