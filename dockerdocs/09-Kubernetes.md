# Docker容器技术学习总结 #

## 9.Kubernetes##
Kubernetes是Google开源的容器集群管理系统。它构建于docker技术之上，为容器化的应用提供资源调度、部署运行、服务发现、扩容缩容等整一套功能，本质上可看作是基于容器技术的mini-PaaS平台。

### 9.1 Kubernetes架构 ###
  



图9-1位Kubernetes的总体概览，基本上可以从操作对象、功能组件和功能特性等三个维度来认识Kubernetes。

![](pics\k8s-frame.jpg)  
图9-1 Kubernetes架构图

#### 9.1.1. Kubernetes操作对象 ####
Kubernetes以RESTFul形式开放接口，用户可操作的REST对象有三个：

- **pod：**是Kubernetes最基本的部署调度单元，可以包含container，逻辑上表示某种应用的一个实例。比如一个web站点应用由前端、后端及数据库构建而成，这三个组件将运行在各自的容器中，那么我们可以创建包含三个container的pod。
- **service：**是pod的路由代理抽象，用于解决pod之间的服务发现问题。因为pod的运行状态可动态变化(比如切换机器了、缩容过程中被终止了等)，所以访问端不能以写死IP的方式去访问该pod提供的服务。service的引入旨在保证pod的动态变化对访问端透明，访问端只需要知道service的地址，由service来提供代理。
- **replicationController（RC）**：是pod的复制抽象，用于解决pod的扩容缩容问题。通常，分布式应用为了性能或高可用性的考虑，需要复制多份资源，并且根据负载情况动态伸缩。通过replicationController，我们可以指定一个应用需要几份复制，Kubernetes将为每份复制创建一个pod，并且保证实际运行pod数量总是与该复制数量相等(例如，当前某个pod宕机时，自动创建新的pod来替换)。


可以看到，service和replicationController只是建立在pod之上的抽象，最终是要作用于pod的，那么它们如何跟pod联系起来呢？这就要引入label的概念。

- **Label:** label是为pod加上可用于搜索或关联的一组key/value标签，而service和replicationController正是通过label来与pod关联的。  

如下图9-2所示，有三个pod都有label为"app=backend"，创建service和replicationController时可以指定同样的label:"app=backend"，再通过label selector机制，就将它们与这三个pod关联起来了。例如，当有其他frontend pod访问该service时，自动会转发到其中的一个backend pod。

![](pics\k8s-pod.jpg)
图9-2 Kubernetes Pod-Label示意图
       

#### 9.1.2 功能组件   

下图9-3是官方文档里的集群架构图，一个典型的master/slave模型。

![](pics\k8s-slavemaster.jpg)    
图9-3 Kubernetes集群架构图---master/slave模型

**master**运行三个组件：

- **apiserver：**作为kubernetes系统的入口，封装了核心对象的增删改查操作，以RESTFul接口方式提供给外部客户和内部组件调用。它维护的REST对象将持久化到etcd（一个分布式强一致性的key/value存储）。
- **scheduler：**负责集群的资源调度，为新建的pod分配机器。这部分工作分出来变成一个组件，意味着可以很方便地替换成其他的调度器。
- **controller-manager：**负责执行各种控制器：  
1）**endpoint-controller：**定期关联service和pod(关联信息由endpoint对象维护)，保证service到pod的映射总是最新的。  
2）**replication-controller：**定期关联replicationController和pod，保证replicationController定义的复制数量与实际运行pod的数量总是一致的。  
3) ...


**slave**(称作minion)运行两个组件：

- **kubelet：**负责管控docker容器，如启动/停止、监控运行状态等。它会定期从etcd获取分配到本机的pod，并根据pod信息启动或停止相应的容器。同时，它也会接收apiserver的HTTP请求，汇报pod的运行状态。
- **proxy：**负责为pod提供代理。它会定期从etcd获取所有的service，并根据service信息创建代理。当某个客户pod要访问其他pod时，访问请求会经过本机proxy做转发。



### 9.2 Label和Label selector

Label标签在Kubernetes模型中占着非常重要的作用。Label表现为key/value对，附加到Kubernetes管理的对象上，典型的就是Pods。它们定义了这些对象的识别属性，用来组织和选择这些对象。Label可以在对象创建时附加在对象上，也可以对象存在时通过API管理对象的Label。

在定义了对象的Label后，其它模型可以用Label 选择器（selector)来定义其作用的对象。

Label选择器有两种，分别是:

- **Equality-based选择器**

		#Equality-based Label selector
		#匹配Label具有environment key且等于production的对象
		environment = production 
		#匹配具有tier key，但是值不等于frontend的对象
		tier != frontend 
		#kubernetes使用AND逻辑，第三条匹配production但不是frontend的对象。
		environment = production，tier != frontend 
- **Set-based选择器**

		#Set-based Label selector
		#选择具有environment key，而且值是production或者qa的label附加的对象
		environment in (production, qa)
		#选择具有tier key，但是其值不是frontend和backend
		tier notin (frontend, backend)
		#选则具有partition key的对象，不对value进行校验
		partition


RC和Service都用label和label selctor来动态地配备作用对象。RC在定义的时候就指定了其要创建Pod的Label和自己要匹配这个Pod的selector， API服务器应该校验这个定义。我们可以动态地修改replication controller创建的Pod的Label用于调式，数据恢复等。一旦某个Pod由于Label改变从Replication Controller移出来后，Replication Controller会马上启动一个新的Pod来确保复制池子中的份数。对于Service，Label selector可以用来选择一个Service的后端Pods。

### 9.3  Kubernetes Service ###
Kubernetes中的Service是一种抽象概念，它定义了一个pods逻辑集合以及访问它们的策略，有时它也被称为微服务（Micro-service）。服务的目标是提供一种桥梁，使得非Kubernetes原生应用程序，在无需为Kubernetes编写特定代码的前提下，轻松访问后端。服务会为用户提供一对IP地址和port端口，用于在访问时重定向到相应的后端。服务里Pods集合的选定是由一个标签选择器（label selector）来完成的。**服务的抽象性实现了前端访问与后端服务的解耦。**

#### 9.3.1 服务定义 ####

A Service in Kubernetes is a REST object, similar to a Pod. Like all of the REST objects, a Service definition can be POSTed to the apiserver to create a new instance. For example, suppose you have a set of Pods that each expose port 9376 and carry a label "app=MyApp".  

	{
    	"kind": "Service",
    	"apiVersion": "v1",
    	"metadata": {
        	"name": "my-service"
   		},
    	"spec": {
        	"selector": {
            	"app": "MyApp"
        	},
        	"ports": [
            	{
                	"protocol": "TCP",
                	"port": 80,
                	"targetPort": 9376
            	}
        	]
    	}
	}
This specification will create a new Service object named “my-service” which targets TCP port 9376 on any Pod with the "app=MyApp" label. This Service will also be assigned an IP address (sometimes called the “cluster IP”), which is used by the service proxies (see below). The Service’s selector will be evaluated continuously and the results will be POSTed to an Endpoints object also named “my-service”.

Note that a Service can map an incoming port to any targetPort. By default the targetPort will be set to the same value as the port field. Perhaps more interesting is that targetPort can be a string, referring to the name of a port in the backend Pods. The actual port number assigned to that name can be different in each backend Pod. This offers a lot of flexibility for deploying and evolving your Services. For example, you can change the port number that pods expose in the next version of your backend software, without breaking clients.
Kubernetes Services support TCP and UDP for protocols. The default is TCP.



9.1.2 Kubernetes工作流程图 

![](pics\k8s-workflow.jpg)