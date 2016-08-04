# Docker容器技术学习总结 #
## 1.容器技术基础 ##
### 1.1 容器的组成 ###
容器的核心技术是Cgroup + Namespace。容器技术主要包括Namespace 和Cgroup 这两个内核特性。
>- Namespace 又称为命名空间（也可翻译为名字空间），它主要做访问隔离。其原理是针对一类资源进行抽象，并将其封装在一起提供给一个容器使用，对于这类资源，因为每个容器都有自己的抽象，而它们彼此之间是不可见的，所以就可以做到访问
隔离。
>- Cgroup 是 control group 的简称，又称为控制组，它主要是做资源控制。其原理是将一组进程放在一个控制组里，通过给这个控制组分配指定的可用资源，达到控制这一组进程可用资源的目的。

实际上，Namespace 和Cgroup 并不是强相关的两种技术，用户可以根据需要单独使用
它们，比如单独使用Cgroup 做资源控制，就是一种比较常见的做法。而如果把它们应用到
一起，在一个Namespace 中的进程恰好又在一个Cgroup 中，那么这些进程就既有访问隔
离，又有资源控制，符合容器的特性，这样就创建了一个容器。

对于Linux 容器的最小组成，可以由以下公式来表示：

>容器 = cgroup + namespace + rootfs + 容器引擎（用户态工具）

目前市场上所有Linux 容器项目都包含以下组件,其中各项的功能分别为：
>- Cgroup：资源控制。
>- Namespace：访问隔离。
>- rootfs：文件系统隔离。
>- 容器引擎：生命周期控制。

### 1.2 Cgroup原理介绍 ###
#### 1.2.1 Cgroup是什么 ####

Details: [RedHat CGroup资源管理指南](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/6/html-single/Resource_Management_Guide/index.html#idp86135408 "RedHat CGroup资源管理指南")

Cgroup 是control group 的简写，属于Linux 内核提供的一个特性，用于限制和隔离一组进程对系统资源的使用，也就是做资源QoS，这些资源主要包括CPU、内存、block I/O和网络带宽。Cgroup 从2.6.24 开始进入内核主线，目前各大发行版都默认打开了Cgroup特性。    
从实现的角度来看，Cgroup 实现了一个通用的进程分组的框架，而不同资源的具体管理则是由各个Cgroup 子系统实现的。截止到内核4.1 版本，Cgroup 中实现的子系统及其作用如下：  

>-  devices：设备权限控制。
-  cpuset：分配指定的 CPU 和内存节点。
-  cpu：控制 CPU 占用率。
-  cpuacct：统计 CPU 使用情况。
-  memory：限制内存的使用上限。
-  freezer：冻结（暂停）Cgroup 中的进程。
-  net_cls：配合 tc（traffic controller）限制网络带宽。
-  net_prio：设置进程的网络流量优先级。
-  huge_tlb：限制 HugeTLB 的使用。
-  perf_event：允许 Perf 工具基于 Cgro

在Cgroup 出现之前，只能对一个进程做一些资源控制，例如通过sched_setaffinity 系统调用限定一个进程的CPU亲和性，或者用ulimit 限制一个进程的打开文件上限、栈大小等。另外，使用ulimit 可以对少数资源基于用户做资源控制，例如限制一个用户能创建的进程数。而Cgroup 可以对进程进行任意的分组，如何分组是用户自定义的，例如安卓的应用分为前台应用和后台应用，前台应用是直接跟用户交互的，需要响应速度快，因此前台应用对资源的申请需要得到更多的保证。为此安卓将前台应用和后台应用划分到不同的Cgroup 中，并且对放置前台应用的Cgroup 配置了较高的系统资源限额。

CGroup有下述术语：

- 任务（Tasks）：就是系统的一个进程。
- 控制组（Control Group）：一组按照某种标准划分的进程，比如官方文档中的Professor和Student，或是WWW和System之类的，其表示了某进程组。Cgroups中的资源控制都是以控制组为单位实现。一个进程可以加入到某个控制组。而资源的限制是定义在这个组上，就像上面示例中我用的haoel一样。简单点说，cgroup的呈现就是一个目录带一系列的可配置文件。
- 层级（Hierarchy）：控制组可以组织成hierarchical的形式，既一颗控制组的树（目录结构）。控制组树上的子节点继承父结点的属性。简单点说，hierarchy就是在一个或多个子系统上的cgroups目录树。
- 子系统（Subsystem）：一个子系统就是一个资源控制器，比如CPU子系统就是控制CPU时间分配的一个控制器。子系统必须附加到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。Cgroup的子系统可以有很多，也在不断增加中。

#### 1.2.2 Cgroup的接口和使用 ####
Cgroup 的原生接口通过cgroupfs 提供，类似于procfs 和sysfs，是一种虚拟文件系统。以下用一个实例演示如何使用Cgroup。  
（1）挂载cgroupfs  
命令如下：  

	#mount –t cgroup –o cpuset cpuset /sys/fs/cgroup/cpuset  
首先必须将cgroupfs 挂载起来，这个动作一般已经在启动时由Linux 发行版做好了。可以把cgroupfs 挂载在任意一个目录上，不过标准的挂载点是/sys/fs/cgroup。

> 实际上sysfs 里面只有/sys/fs/cgroup 目录，并且sysfs 是不允 许用户创建目录的，这
里可以将tmpfs 挂载上去，然后在tmpfs 上创建目录

（2）查看cgroupfs  

	 #ls /sys/fs/cgroup/cpuset  
	cgroup.clone_children cpuset.memory_pressure  
	cgroup.procs cpuset.memory_pressure_enabled  
	cgroup.sane_behavior cpuset.memory_spread_page  
	cpuset.cpu_exclusive cpuset.memory_spread_slab
	cpuset.cpus cpuset.mems  
	cpuset.effective_cpus cpuset.sched_load_balance  
	cpuset.effective_mems cpuset.sched_relax_domain_level  
	cpuset.mem_exclusive notify_on_release  
	cpuset.mem_hardwall release_agent  
	cpuset.memory_migrate  

可以看到这里有很多控制文件，其中以cpuset 开头的控制文件都是cpuset 子系统产生的，其他文件则由Cgroup 产生。这里面的tasks 文件记录了这个Cgroup 的所有进程（包括线程），在系统启动后，默认没有对Cgroup 做任何配置的情况下，cgroupfs 只有一个根目录，并且系统所有的进程都在这个根目录中，即进程pid 都在根目录的tasks 文件中。

> 实际上现在大多数Linux发行版都是采用的systemd，而systemd 使用了Cgroup，所以在这些发行版上，当系统启动后看到的cgroupfs是有一些子目录的。

（3）创建Cgroup  

	#mkdir /sys/fs/cgroup/cpuset/child
通过mkdir 创建一个新的目录，也就创建了一个新的Cgroup。

（4）配置Cgroup

	#echo 0 > /sys/fs/cgroup/cpuset/child/cpuset.cpus  
	#echo 0 > /sys/fs/cgroup/cpsuet/child/cpuset.mems  
接下来配置这个Cgroup 的资源配额，通过上面的命令，就可以限制这个Cgroup的进程只能在0号CPU上运行，并且只会从0号内存节点分配内存。

（5）使能Cgroup  

	#echo $$ > /sys/fs/cgroup/cpuset/child/tasks  
最后，通过将进程id 写入tasks 文件，就可以把进程移动到这个Cgroup 中。并且，这个进程产生的所有子进程也都会自动放在这个Cgroup 里。这时，Cgroup 才真正起作用了。
> $$ 表示当前进程。另外，也可以把pid 写入cgroup.procs 中，两者的区别是写入tasks 只会把指定的进程加到child 中，写入cgroup.procs 则会把这个进程所属的整个线程组都加到child 中。

#### 1.2.3 Cgroup子系统介绍 ####
对实际资源的分配和管理是由各个Cgroup子系统完成的，下面介绍几个主要的子系统。 
 
**1. cpuset 子系统**   
cpuset 可以为一组进程分配指定的CPU和内存节点。cpuset一开始是用在高性能计算（HPC）上的，在NUMA 架构的服务器上，通过将进程绑定到固定的CPU 和内存节点上，来避免进程在运行时因跨节点内存访问而导致的性能下降。当然，现在cpuset 也广泛用在了kvm 和容器等场景上。  
cpuset 的主要接口如下。  

- cpuset.cpus：允许进程使用的 CPU 列表（例如 0 ～ 4,9）。  
- cpuset.mems：允许进程使用的内存节点列表（例如 0 ～ 1）。

**2. cpu 子系统**  
cpu 子系统用于限制进程的CPU 占用率。实际上它有三个功能，分别通过不同的接口来提供。
  
- CPU 比重分配。这个特性使用的接口是 cpu.shares。假设在 cgroupfs 的根目录下创建了两个Cgroup（C1 和C2），并且将cpu.shares 分别配置为512 和1024，那么当C1 和C2 争用CPU 时，C2 将会比C1 得到多一倍的CPU 占用率。要注意的是，只有当它们争用CPU 时cpu share 才会起作用，如果C2 是空闲的，那么C1 可以得到全部的CPU 资源。
- CPU 带宽限制。这个特性使用的接口是 cpu.cfs\_period\_us 和 cpu.cfs\_quota\_us，这两个接口的单位是微秒。可以将period 设置为1 秒，将quota 设置为0.5 秒，那么Cgroup 中的进程在1 秒内最多只能运行0.5 秒，然后就会被强制睡眠，直到进入下一个1 秒才能继续运行。
- 实时进程的 CPU 带宽限制。以上两个特性都只能限制普通进程，若要限制实时进程，就要使用cpu.rt\_period\_us 和cpu.rt\_runtime\_us 这两个接口了。使用方法和上面类似。

**3. cpuacct 子系统**  
cpuacct 子系统用来统计各个Cgroup 的CPU 使用情况，有如下接口。  

- cpuacct.stat ：报告这个 Cgroup 分别在用户态和内核态消耗的 CPU 时间，单位是USER_HZ。USER\_HZ 在x86 上一般是100，即1 USER\_HZ 等于0.01 秒。
- cpuacct.usage：报告这个 Cgroup 消耗的总 CPU 时间，单位是纳秒。  
- cpuacct.usage\_percpu ：报告这个 Cgroup 在各个 CPU 上消耗的 CPU 时间，总和也就是cpuacct.usage 的值。

**4. memory 子系统**  
memory 子系统用来限制Cgroup 所能使用的内存上限，有如下接口。 
 
- memory.limit_in_bytes：设定内存上限，单位是字节，也可以使用 k/K、m/M 或者 g/G 表示要设置数值的单位，例如   
	`#echo 1G > memory.limit_in_bytes`
> 默认情况下，如果Cgroup 使用的内存超过上限，Linux 内核会尝试回收内存，如果仍然无法将内存使用量控制在上限之内，系统将会触发OOM，选择并“杀死”该Cgroup 中的某个进程。

- memory.memsw.limit\_in_bytes ：设定内存加上交换分区的使用总量。通过设置这个值，可以防止进程把交换分区用光。
- memory.oom\_control：如果设置为 0，那么在内存使用量超过上限时，系统不会“杀死”进程，而是阻塞进程直到有内存被释放可供使用时；另一方面，系统会向用户态发送事件通知，用户态的监控程序可以根据该事件来做相应的处理，例如提高内
存上限等。
- memory.stat：汇报内存使用信息。

**5. blkio 子系统**  
blkio 子系统用来限制Cgroup 的block I/O 带宽，有如下接口。  

- blkio.weight ：设置权重值，范围在 100 到 1000 之间。这跟 cpu.shares 类似，是比重分配，而不是绝对带宽的限制，因此只有当不同的Cgroup 在争用同一个块设备的带宽时才会起作用。
- blkio.weight\_device：对具体的设备设置权重值，这个值会覆盖上述的 blkio.weight。  
例如将Cgroup 对/dev/sda 的权重设为最小值：
	`#echo 8:0 100 > blkio.weight_device`
- blkio.throttle.read_bps_device：对具体的设备，设置每秒读磁盘的带宽上限。例如对/dev/sda 的读带宽限制在1MB/ 秒：`#echo "8:0 1048576" > blkio.throttle.read_bps_device`  
- blkio.throttle.write\_bps\_device：设置每秒写磁盘的带宽上限。同样需要指定设备。
- blkio.throttle.read\_iops\_device：设置每秒读磁盘的 IOPS 上限。同样需要指定设备。
- blkio.throttle.write\_iops\_device：设置每秒写磁盘的 IOPS 上限。同样需要指定设备。
> blkio 子系统有两个重大的缺陷。一是对于比重分配，只支持CFQ 调度器。另一个缺陷是，不管是比重分配还是上限限制，都只支持Direct-IO，不支持Buffered-IO，这使得blkio cgroup 的使用场景很有限，好消息是Linux 内核社区正在解决这个问题。

**6. devices 子系统**  
devices 子系统用来控制Cgroup 的进程对哪些设备有访问权限，有如下接口。

- devices.list。只读文件，显示目前允许被访问的设备列表。每个条目都有3个域，分别为：  
类型：可以是 a、c 或 b，分别表示所有设备、字符设备和块设备。  
设备号：格式为 major:minor 的设备号。  
权限：r、w 和 m 的组合，分别表示可读、可写、可创建设备结点（mknod）。  
>>例如“ a *:* rmw”，表示所有设备都可以被访问。而“ c 1:3 r”，表示对1:3 这个字符
设备（即/dev/null）只有读权限。

- devices.allow。只写文件，以上面描述的格式写入该文件，就可以允许相应的设备访
问权限。
- devices.deny。只写文件，以上面描述的格式写入该文件，就可以禁止相应的设备访
问权限。

### 1.3 NameSpace介绍 ###
#### 1.3.1 NameSpace是什么 ####
Namespace 是将内核的全局资源做封装，使得每个Namespace 都有一份独立的资源，因此不同的进程在各自的Namespace 内对同一种资源的使用不会互相干扰。  
> 举个例子，执行sethostname 这个系统调用时，可以改变系统的主机名，这个主机名就是一个内核的全局资源。内核通过实现UTS Namespace，可以将不同的进程分隔在不同的UTS Namespace 中，在某个Namespace 修改主机名时，另一个Namespace 的主机名还是保持不变。
目前Linux 内核总共实现了6 种Namespace：

- IPC：隔离 System V IPC 和 POSIX 消息队列。
- Network：隔离网络资源。
- Mount：隔离文件系统挂载点。
- PID：隔离进程 ID。
- UTS：隔离主机名和域名。
- User：隔离用户 ID 和组 ID。

#### 1.3.2 Namespace的接口和使用 ####
对Namespace 的操作，主要是通过clone、setns 和unshare 这3 个系统调用来完成的。 
 
clone 可以用来创建新的Namespace。它接受一个叫flags 的参数， 这些flag 包括CLONE\_NEWNS、CLONE\_NEWIPC、CLONE\_NEWUTS、CLONE\_NEWNET、CLONE\_NEWPID 和CLONE\_NEWUSER，我们通过传入这些CLONE\_NEW 来创建新的Namespace。这些flag 对应的Namespace 都可以从字面上看出来，除了CLONE_NEWNS，这是用来创建Mount Namespace 的。指定了这些flag 后，由clone 创建出来的新进程，就位于全新的Namespace 里了，并且很自然地这个新进程以后创建出来的进程，也都在这个Namespace 中。
> Mount Namespace 是第一个实现的Namespace，当初实现时并不是为了实现Linux容器，因此也就没有预料到会有新的Namespace 出现，因此用了CLONE_NEWNS而不是CLONE_NEWMNT 之类的名字。

那么，能不能为已有的进程创建新的Namespace 呢？答案是可以，unshare 就是用来达到这个目的的。调用这个系统调用的进程，会被放进新创建的Namespace 里，要创建什么Namespace 由flags 参数指定，可以使用的flag 也就是上面提到的那些。  

以上两个系统调用都是用来创建新的Namespace的，而setns 则可以将进程放到已有的Namespace 里，问题是如何指定已有的Namespace？答案在procfs 里。每个进程在procfs下都有一个目录，在那里面就有Namespace 相关的信息，如下。
	
	#ls –l /proc/$$/ns
	total 0
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 ipc -> ipc:[4026531839]
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 mnt -> mnt:[4026531840]
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 net -> net:[4026531957]
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 pid -> pid:[4026531836]
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 user -> user:[4026531837]
	lrwxrwxrwx 1 root root 0 Jun 16 14:39 uts -> uts:[4026531838]
这里每个虚拟文件都对应了这个进程所处的Namespace。因此，如果另一个进程要进入这个进程的Namespace，可以通过open系统调用打开这里面的虚拟文件并得到一个文件描述符，然后把文件描述符传给setns，调用返回成功的话，就进入这个进程的Namespace 了。
> docker exec 命令的实现原理就是setns。

以下是一个简单的程序，在Linux 终端调用这个程序就会进入新的Namespace，同
时也可以打开另一个终端，这个终端是在host 的Namespace 里，这样就可以对比两个
Namespace 的区别了。  

	#define _GNU_SOURCE
	#include <sys/wait.h>
	#include <sys/utsname.h>
	#include <sched.h>
	#include <stdio.h>
	#define STACK_SIZE (1024 * 1024)**
	static char stack[STACK_SIZE];
	static char* const child_args[] = { "/bin/bash", NULL };
	static int child(void *arg)
	{
		execv("/bin/bash", child_args);
		return 0;
	}
	int main(int argc, char *argv[])
	{
		pid_t pid;
		pid = clone(child, stack+STACK_SIZE, SIGCHLD|CLONE_NEWUTS, NULL);
		waitpid(pid, NULL, 0);
	}
这个程序创建了UTS Namespace，可以通过修改flag，创建其他Namespace，也可以创建几个Namespace 的组合。这个程序将会用来为下面的内容做演示。

#### 1.3.3 各个Namespace介绍 ####
**1. UTS Namespace**  
UTS Namespace 用于对主机名和域名进行隔离，也就是uname系统调用使用的结构体struct utsname 里的nodename 和domainname 这两个字段，UTS 这个名字也是由此而来的。*那么，为什么要使用UTS Namespace 做隔离？这是因为主机名可以用来代替IP 地址，因此，也就可以使用主机名在网络上访问某台机器了，如果不做隔离，这个机制在容器里就会出问题。*

调用之前的程序后，在Namespace 终端执行以下命令：

	# hostname container
	# hostname
	container
这里已经改变了主机名，现在通过host 终端来看看host 的主机名：

	# hostname
	linux-host
可以看到，host 的主机名并没有变化，这就是Namespace 所起的作用。

**2. IPC Namespace**  
IPC 是Inter-Process Communication 的简写，也就是进程间通信。Linux 提供了很多种进程间通信的机制，IPC Namespace 针对的是SystemV IPC 和Posix 消息队列。这些IPC 机制都会用到标识符，例如用标识符来区别不同的消息队列，然后两个进程通过标识符找到对应的消息队列进行通信等。  
IPC Namespace 能做到的事情是，使相同的标识符在两个Namespace 中代表不同的消息队列，这样也就使得两个Namespace 中的进程不能通过IPC 进程通信了。  
举个例子，在namespace 终端创建了一个消息队列：

	# ipcmk -Q
	Message queue id: 65536
	# ipcs -q  
	------ Message Queues --------
	key msqid owner perms used-bytes messages
	0x0ec037c7 65536 root 644 0
这个消息队列的标识符是65536，现在在host 终端看一下：

	# ipcs -q
	------ Message Queues --------
	key msqid owner perms used-bytes messages
在这里看不到任何消息队列，IPC 隔离的效果达到了。

**3. PID Namespace**  
PID Namespace 用于隔离进程PID 号，这样一来，不同的Namespace 里的进程PID 号就可以是一样的了。  
当创建一个PID Namespace 时，第一个进程的PID 号是1，也就是init 进程。init 进程有一些特殊之处，例如init 进程需要负责回收所有孤儿进程的资源。另外，发送给init 进程的任何信号都会被屏蔽，即使发送的是SIGKILL信号，也就是说，在容器内无法“杀死”init 进程。  
但是当用ps 命令查看系统的进程时，会发现竟然可以看到host 的所有进程：

	# ps ax
	PID TTY STAT TIME COMMAND
	1 ? Ss 0:24 init [5]
	2 ? S 0:06 [kthreadd]
	3 ? S 1:37 [ksoftirqd/0]
	5 ? S< 0:00 [kworker/0:0H]
	7 ? S 0:16 [kworker/u33:0]
	...
	7585 pts/0 S+ 0:00 sleep 1000
这是因为ps 命令是从procfs 读取信息的，而procfs 并没有得到隔离。虽然能看到这些进程，但由于它们其实是在另一个PID Namespace 中，因此无法向这些进程发送信号：

	# kill -9 7585
	-bash: kill: (7585) - No such process

**4. Mount Namespace**  
Mount Namespace 用来隔离文件系统挂载点，每个进程能看到的文件系统都记录在/proc/$$/mounts 里。在创建了一个新的Mount Namespace 后，进程系统对文件系统挂载/卸载的动作就不会影响到其他Namespace。  
之前看到，创建PID Namespace 后，由于procfs 没有改变，因此通过ps 命令看到的仍然是host 的进程树，其实可以通过在这个PID Namespace 里挂载procfs 来解决这个问题，
如下：

	# mount –t proc none /proc
	# ps ax
	PID TTY STAT TIME COMMAND
	1 pts/2 S+ 0:00 newns
	3 pts/2 R+ 0:00 ps ax
但此时由于文件系统挂载点没有隔离，因此host 看到的procfs 也会是这个新的procfs，
这样在host 上就会出问题：

	# ps ax
	Error, do this: mount -t proc none /proc
可如果同时使用Mount Namespace 和PID Namespace， 新的Namespace 里的进程和host 上的进程将会看到各自的procfs，故而也就不存在上面的问题了。

**5. Network Namespace**  
这个Namespace 会对网络相关的系统资源进行隔离，每个Network Namespace 都有自己的网络设备、IP 地址、路由表、/proc/net 目录、端口号等。网络隔离的必要性是很明显的，举一个例子，在没有隔离的情况下，如果两个不同的容器都想运行同一个Web 应用，而这个应用又需要使用80 端口，那就会有冲突了。

新创建的Network Namespace 会有一个loopback 设备，除此之外不会有任何其他网络设备，因此用户需要在这里面做自己的网络配置。IP 工具已经支持Network Namespace，可以通过它来为新的Network Namespace 配置网络功能。首先创建Network Namespace：

	# ip netns add new_ns
使用“ip netns exec”命令可以对特定的Namespace 执行网络管理：

	# ip netns exec new_ns ip link list
	1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT
	   link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
看到确实只有loopback 这个网络接口，并且它还因处于DOWN 状态而不可用：

	# ip netns exec new_ns ping 127.0.0.1
	connect: Network is unreachable
通过以下命令可以启用loopback 网络接口：

	# ip netns exec new-ns ip link set dev lo up
	# ip netns exec new-ns ping 127.0.0.1
	PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
	64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.053 ms
	...
最后可以这样删除Namespace：

	# ip netns delete new_ns
容器的网络配置是一个很大的话题，后面有专门的章节讲解，因此这里暂不展开

**6. User Namespace**  
User Namespace 用来隔离用户和组ID，也就是说一个进程在Namespace 里的用户和组ID 与它在host 里的ID 可以不一样，这样说可能读者还不理解有什么实际的用处。User Namespace 最有用的地方在于，host 的普通用户进程在容器里可以是0 号用户，也就是root用户。这样，进程在容器内可以做各种特权操作，但是它的特权被限定在容器内，离开了这个容器它就只有普通用户的权限了。
>注意：容器内的这类root 用户，实际上还是有很多特权操作不能执行，基本上如果这个特
权操作会影响到其他容器或者host，就不会被允许。
在host 上，可以看到我们是lizf 用户。

	$ id
	uid=1000(lizf) gid=100(users) groups=100(users)
现在创建新的User Namespace，看看又是什么情况？

	$ new-userns
	$ id
	uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
可以看到，用户名和组名都变了，变成65534，不再是原来的1000 和100。接下来的问题是，怎么设定Namespace 和host 的UID 的映射关系？方法是在创建新的Namespace 后，设置这个Namespace 里进程的/proc/<PID>/uid_map。在Namespace 终端看到的是这样的：

	$ id
	uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
	$ echo $$
	17074
	$ cat /proc/17074/uid_map
	$
可以看到uid_map 是空的，也就是还没有UID 的映射。这可以在host 终端上通过root用户设置，如下。
	# echo "0 1000 65536" > /proc/17074/uid_map
上面命令表示要将[1000, 66536] 的UID 在Namespace 里映射成[0, 65536]。再切回到Namespace 终端看看：
$ id
	uid=0(root) gid=65534(nogroup) 65534(nogroup)
可以看到，我们成功地将lizf 用户映射成容器里的root 用户了。对于gid，也可以做类似的操作。  

----------
"容器造就Docker"
----------

Namespace 和Cgroup的使用是很灵活的，同时这里面又有不少需要注意的地方，因此直接操作Namespace 和Cgroup 并不是很容易。正是因为这些原因，Docker 通过Libcontainer 来处理这些底层的事情。这样一来，Docker 只需要简单地调用Libcontainer 的API，就能将完整的容器搭建起
来。而作为Docker 的用户，就更不用操心这些事情了，而只需要学习Docker 的使用手册，就能通过一两条简单的Docker 命令启动容器。

### 1.4 容器的创建原理 ###
至此对容器的描述还一直停留在文件和概念的层面，本小节将通过简单的代码抽象，清晰地展现容器的创建原理，使读者对容器有更深刻的理解。  
代码一：  

	#pid = clone(fun, stack, fl ags, clone_arg);  
	(fl ags: CLONE_NEWPID | CLONE_NEWNS |  
	CLONE_NEWUSER | CLONE_NEWNET |  
	CLONE_NEWIPC | CLONE_NEWUTS |  
	…)  
    
代码二：  

	#echo $pid > /sys/fs/cgroup/cpu/tasks  
	#echo $pid > /sys/fs/cgroup/cpuset/tasks  
	#echo $pid > /sys/fs/cgroup/blkio/tasks  
	#echo $pid > /sys/fs/cgroup/memory/tasks  
	#echo $pid > /sys/fs/cgroup/devices/tasks  
	#echo $pid > /sys/fs/cgroup/freezer/tasks  
代码三：

	fun()  
	{  
	…  
	pivot_root("path_of_rootfs/", path);  
	…  
	exec("/bin/bash");  
	…  
	}  

- 对于代码一，通过 clone 系统调用，并传入各个 Namespace 对应的 clone flag，创建
了一个新的子进程，该进程拥有自己的Namespace。根据以上代码可知，该进程拥
有自己的pid、mount、user、net、ipc、uts namespace。  
- 对于代码二，将代码一中产生的进程 pid 写入各个 Cgroup 子系统中，这样该进程就
可以受到相应Cgroup 子系统的控制。  
- 对于代码三，该 fun 函数由上面生成的新进程执行，在 fun 函数中，通过 pivot_
root 系统调用， 使进程进入一个新的rootfs， 之后通过exec 系统调用， 在新的
Namespace、Cgroup、rootfs 中执行“/bin/bash”程序。
通过以上操作， 成功地在一个“容器” 中运行了一个bash 程序。


