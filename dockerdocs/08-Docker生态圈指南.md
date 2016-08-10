# Docker容器技术学习总结 #

## 8.Docker生态圈指南##
参考：  
[https://www.mindmeister.com/zh/389671722/docker-ecosystem](https://www.mindmeister.com/zh/389671722/docker-ecosystem)  
[http://blog.dennybritz.com/2015/10/01/a-brief-guide-to-the-docker-ecosystem/](http://blog.dennybritz.com/2015/10/01/a-brief-guide-to-the-docker-ecosystem/)
![](pics\Docker-Ecosystem-v0.77.png)

 
**1. 引擎 / 运行环境**  

容器引擎是容器技术的核心。 引擎用来创建和运行容器。 谈论 Docker 时，一般指的就是 Docker 引擎。

- Docker Engine: 是当前最流行的引擎，也是事实的工业标准。
- rkt: CoreOS 团队主导的开源引擎，用于替换 Docker 引擎。

**2. 支持 Docker 的云服务商**  

在云主机上安装 Docker 运行容器，当然没有任何问题。不过，云服务商的容器服务，能提供更简洁友好的管理工具。

- Amazon EC2 Container Service: 在 EC2 实例上运行容器服务。容器服务免费，只需要支付 EC2 费用。
- Google Container Engine: 构建于 Kubernetes 之上。Google 提供。
- Azure: 机遇 Mesos。Microsoft 提供。
- Stackdock: 提供 Docker 容器托管。
- Tutum: 提供 Docker 容器托管。
- GiantSwarm: 是一家云平台，提供容器中运行的微服务架构的定制和托管。
- Joyent Triton: 提供 Docker 容器监控和托管。
- Jelastic Docker

**3. 容器封装工具**  

封装工具现在是最热闹的领域。 管理少数几个容器很简单，但是管理大量容器保持高扩展性却很有挑战。
容器封装工具解决了很多繁琐的问题：

- 容器资源分配
- 容错
- 共享存储
- 负载均衡
- 网络拓扑管理

常见的项目有：

- Kubernetes: Google 开源的工具，在功能特性方面是当前最先进的工具。
- Docker Swarm: 与 Docker 环境紧密集成。
- Rancher: 以 stack(linked containers) 为单位管理容器。有直观的界面和良好的文档。
- Mesosphere: 通用的数据中心管理系统。不是专为 Docker 开发，能轻松管理容器，也可以与其它系统如 Kubernetes 集成。
- CoreOS fleet: CoreOS 的一部分，负责在 CoreOS 集群中调度容器。
- Nomad: 通用的应用调度工具，内置支持 Docker。
- Centurion: Newrelic 的内部部署工具。
- Flocker: 容器间数据的管理工具。
- Weave Run: 提供服务发现、路由、负载均衡和微服务架构的地址管理。

**4. 操作系统**  

几乎任何操作系统都可以用来运行容器。为运行容器，一般使用最小化的系统镜像。 很多公司提供包含多种实用工具的镜像。

- CoreOS
- Project Atomic: Docker / Kubernetes / rpm / systemd。
- Rancher OS: 只有 20MB 大小。区分 系统容器 和 用户容器。
- Project Photon: VMWare 开源的工具。

**5. 容器镜像仓库 Registry**  

Registry 实现和服务商有：

- Docker Registry: 最流行的开源实现。
- DockerHub: 公开的 Registry 服务。
- Quay.io: CoreOS 开发的容器仓库。
- CoreOS Enterprise Registry

**6. 监控**  

容器输出的日志，很方便与已有日志收集工具整合。容器监控一般关注资源使用（CPU，内存）。

- cAdvisor Google: 开源项目。分析资源使用和性能特性，可以用 InfluxDB 作为数据存储，以便后续分析。
- Datadog Docker: 收集容器的运行信息，发送到 Datadog 分析。
- NewRelic Docker: 发送容器统计信息到 NewRelic 的云服务。
- Sysdig: 监控容器资源使用情况。
- Weave Scope: 自动生成容器关系图，有助于理解、监控和控制应用服务。
- AppFormix: 实时基础设施监控，支持 Docker 容器。