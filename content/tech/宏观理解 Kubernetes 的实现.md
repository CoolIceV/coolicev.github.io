---
title: <div class="custom-title">宏观理解&nbsp;<div class='reverse'>Kubernetes</div>&nbsp;的实现</div>
date: "2022-12-15T14:45:15+08:00"
tags: ["Cloud Native", "Kubernetes", "云原生"]
slug: "understand-kubernetes-in-top-view"
preface: "{{<quote >}}这是我一年前在字节跳动实习时写的一篇学习笔记，简要总结了《Kubernetes in Action》中的部分内容，最近对其进行了重新整理，并计划补充 CRD、Informer、Helm、KubeFed、HNC 等内容。虽然文章讲解的不算深入，但对于想要在相对宏观上了解 Kubernetes 的人来说还是可以一读的。本讲作为云原生系列文章的第一篇，也是本博客的第一篇技术文章，希望能够开一个好头，完成之后还打算在本系列中添加边缘计算、服务网格、函数计算、持续交付、混沌工程等相关内容的介绍。{{</quote >}}"
indentFirstParagraph: true
indent: false
dropCapAfterHr: true
dropCap: true
toc: false
leftTOC: false
# draft: true
---

---

云原生可以说是近年来最火的概念之一，云原生并不是某种具体的技术，而是一种应用设计思想，即应用程序原生为云设计，从设计之初就考虑云的环境，充分发挥云平台的弹性和分布式优势。在云原生领域中，Kubernetes 可以说是目前具有统治地位的一个工具，已成为云原生领域开发的事实标准，也正因为这个工具，越来越多的企业才能够以较低的成本搭建和维护私有云环境，实现应用的持续性交付，加速软件的迭代升级。因此，理解 Kubernetes 十分重要，属于云原生系列文章的基础，后面的内容也基本都会基于 Kubernetes 框架介绍。

本文我们更注重 Kubernetes 的核心概念，从宏观上理解 Kubernetes 的实现思想，而不是如何使用 Kubernetes，自然也不会过分的关注具体细节。

## 一、什么是 Kubernetes

Kubernetes 是一个容器应用的分布式自动化管理平台，目前已经广泛应用在生产环境中，是近几年最热门的的技术之一。Kubernetes 是希腊语『舵手』的意思，它是 Google 实现的 Borg 的开源方案，由于 K 和 s 之间有 8 个字符，一般被缩写为 K8s。

![Kubernetes](/images/宏观理解Kubernetes的实现/k8s-intro.svg "Kubernetes")

K8s 相当于云原生时代的“操作系统”，它部署在集群之上的，在底层基础架构和设施之上提供了一个具有海量资源与算力的抽象层，使数以千计的计算机节点相互协调运作，使用容器技术和微服务架构对应用进行自动化编排与调度。开发人员通过 K8s 部署应用或服务时无需关心底层计算机和基础架构，更不必手动到每台机器上操作，极大程度上简化了服务的部署运维流程。更重要的是，K8s 具有服务、节点健康检查与自修复功能，当服务/节点出现故障之时，可以自动在可用节点上重建服务，使整个系统所提供的服务仍处于期望的状态。

### Kubernetes 的两大相关技术

K8s 之所以能够逐渐被广泛应用，与容器技术和微服务架构的迅速发展离不开关系，我们在这里简单介绍一下这两大技术：

#### 第一个技术：容器技术

容器技术让开发人员可以将自己应用和依赖环境打包到一个可移植的容器中，让应用的运行与环境无关。容器与虚拟机类似，每个容器具有自己的文件系统、进程空间以及CPU和内存限制等，不同的是，容器并没与像虚拟机一样虚拟出硬件与内核，而是使用宿主机的硬件与内核，因此，容器会比虚拟机更轻量，资源利用率也更高。下面这张图比较清晰得展示了虚拟机和容器的最大的区别，可以看到容器是比较轻量的，调用流程也更简单，不会像虚拟机一样占用过多的资源。

![VM 与 Container 对比](/images/宏观理解Kubernetes的实现/container.svg "VM 与 Container 对比")

Docker 可以说是当前容器技术的标准，在多数生产环境中的云环境也都是使用的 Docker 作为运行环境。但是事实上，在 2020 年 12 月，K8s 社区决定移除仓库中支持 Docker 的代码，主要原因是 Docker 并没有实现 K8s 需要的 CRI（容器运行时接口，Container Runtime Interface），之前都是由 K8s 社区来实现和维护一个叫 dockershim 的组件，对于今天已经统治市场的 K8s 来说，支持 Docker 的维护成本过于昂贵，而且也不符合 K8s 高扩展性的发展目标，因此不再进行维护。但现在 Docker 已经使用 cri-dockerd 支持了 K8s 所需要的接口。

#### 第二个技术：微服务架构
随着现在软件越来越复杂，一个完整的应用不再是单体服务，而是被拆分成多个核心功能，每个功能都被称为一项服务，具有特定的职责和功能，可以单独构建和部署，这也就意味着各个服务在工作甚至出现故障时不会相互影响，这种应用的实现方式被称为微服务架构。但微服务个数的增多意味着链路的延长，响应的时延也会随之增大，为了降低处理时延，微服务架构又出现了合并部署的趋势，将相关联的服务在云环境中部署的尽可能靠近，并利用 Service Mesh 技术进行流量调度，将微服务进行“合并”。

![服务架构](/images/宏观理解Kubernetes的实现/service-arch.svg "服务架构")

### Kubernetes 的三大核心能力

#### 第一个核心能力：编排调度

K8s 可以把服务资源请求与实际剩余资源进行自动编排分配，将服务实例分配到集群的具体节点上，用户无需感知被分配到了哪台机器上，这种自动化编排的方式除了能够减少人力还能有效提升资源利用率。此外，K8s 还会对服务的全生命周期进行监控管理控制，极大程度上简化了操作。

#### 第二个核心能力：自动修复

K8s 可以对节点甚至容器进行健康检查，当有一个节点出现问题时，K8s 会将这个节点上的所有容器自动迁移到其他节点上，如果容器出现了问题会对这个容器进行重启，来实现自动修复的功能。不止节点和容器，任何其他资源甚至自定义的资源都能够很容易实现自动修复的逻辑。

#### 第三个核心能力：弹性伸缩

K8s 有业务负载检查的功能，它会监测业务上所承担的负载，并根据负载情况对服务副本个数甚至服务资源配置进行弹性调整。K8s 为我们提供了一个能够自动进行扩缩容的对象 HPA，HPA 会针对每一个指标使用所有容器的平均值除以目标值得到应有的副本数，然后所有指标计算出的结果中取最大值，根据这个值对服务实例进行水平扩缩容。

![HPA](/images/宏观理解Kubernetes的实现/hpa.svg "HPA")

HPA 是支持一些更高级的配置的，比如最大的副本数、最小的副本数、每次扩缩容影响实例个数的比例等，还可以使用自定义指标，具体的可以查看 [Pod 水平自动扩缩 | Kubernetes](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/)。


## 二、Kubernetes 的管理

我们对 K8s 集群的管理都是通过客户端调用 K8s 提供的 API 来进行管理，最常用的一个工具是 kubectl，它是一个命令行工具，利用 K8s 的 API 整合出了一系列功能强大的命令，也就是说，运行一个 kubectl 命令在背后可能是调用了数个原生 API。我们并不介绍如何使用 kubectl，因为官方的文档和网络上的相关博客已经介绍的十分全面了，想要实现什么操作在网络上都可以找到命令及示例，感兴趣的可以 [点击学习如何使用kubectl](https://kubernetes.io/zh/docs/reference/kubectl/)。

接下来我们来看一些与管理相关的比较“抽象”的内容。

### 第一个概念：对象

K8s 中的对象是系统中持久化的实体，它们表示的是集群的状态，有的地方也被称为“资源”，实际上对象应该是资源的实例化，并不直接等同。这些对象描述了哪些应用或服务应该以怎样的规格运行在集群中，每对一个对象进行增删改查操作，都是对集群状态的修改。K8s 使用声明式的方式定义对象，我们并不告诉 K8s 具体要干什么，只告诉 K8s 我们期望的一个稳定的状态，然后K8s会进行编排、管理来达到这个状态。

接下来介绍一下怎么描述一个具体的 K8s 对象以及它是以什么形式存储的。

**如何描述**

我们可以使用 JSON 或 YAML 格式的数据来描述一个对象，每一个对象都包括以下四个部分：
- `Type Metadata`：API 组和类型，这一部分包含了这个对象的类型，K8s 的对象是根据 API 版本分组进行管理的，不同类型的资源可能在不同的组或版本下，我们通过这一部分明确的告诉 K8s 对象的类型。
- `Object Metadata`：元数据，这一部分描述的是对象的基本信息，包括对象名称、创建时间、标签、命名空间和其他来标识对象的信息，我们获取对象的时候是根据这部分的信息中的名称、命名空间、标签等进行匹配的。

以上两部分对于所有对象来说字段都是相同的，而接下来两个字段不同类型的对象拥有着不同的定义。

- `Spec`：对象的规格，这部分详细描述了对象的内容，是我们期望的对象的状态，里面包含了该对象需要的资源以及策略，K8s 的调度其实就是不断进行操作来使集群达到或处于这部分的状态。
- `Status`：对象当前的状态，这部分描述的是对象在 K8s 中某一具体时间的实际状态，这一部分会表明 K8s 实际为该对象分配的资源以及该对象所处的生命周期阶段，同时还会包括该对象最近产生的变化以及对应的原因等信息，K8s 会一直监听该对象来使该对象的实际状态与 Spec 描述的一致。

![Kubernetes 对象](/images/宏观理解Kubernetes的实现/object.svg "Kubernetes 对象")


**如何存储**

我们再来说一下 K8s 是怎么存储对象的，K8s 会将所有的对象存储在 etcd（分布式统一键值存储）中，对象的键是 `/registry/{对象类型}/{对象命名空间}/{对象名称}`，而对象的值就是我们刚刚介绍的这些内容的 json 格式，在比较旧的 K8s 版本中是明文存储的，1.7 版本之后 K8s 对内容进行了加密。
```shell
$ etcdctl get /registry/pods/default/kubia-manual 
{"kind":"Pod","apiVersion":"v1","metadata":{"name":"kubia-manual", ...
```

### 第二个概念：标签
在 K8s 中，最基本也是最重要的对象是一种被称为Pod 的资源，Pod 内部会运行若干个容器来提供服务，每个服务的一个实例（副本）就是一个 Pod，我们对一个服务的最基本的管理就是对 Pod 的管理。

微服务架构的系统会有大量的服务部署在 K8s 上，每个服务又可能会同时部署了多个版本，每个版本又具有多个副本，这就会导致系统中会有着数千甚至上万个 Pod，而且这些 Pod 又是“混乱”得分布在各个节点上，我们不可能一个 Pod 一个 Pod 得进行管理，我们肯定更希望能够将对其中一个子集进行（批量）管理，K8s 使用标签为我们提供了一种简单但又十分强大的管理功能。

![Kubernetes Pod](/images/宏观理解Kubernetes的实现/label-1.svg "Kubernetes Pod")

标签是 K8s 对象的一个十分重要的特性，一个标签就是一个的 k-v 键值对，我们可以为 K8s 的任意对象添加标签，之后我们便可以通过标签筛选出一个集合的对象来进行管理，标签的添加或修改可以在对象创建之前也可以在创建之后，这为我们提供了比较大的灵活性。

我们为上图所示的所有 Pod 都添加两个标签，一个是 *app*，代表这个 Pod 属于哪个服务，还有一个是 *rel*，表示这个 Pod 运行的服务是稳定版还是体验版或者小流量版。添加完成之后，我们就为这些 Pod 建立起了一个二维的管理空间，如下图所示，我们可以通过`rel=canary`选中所有小流量版本的 Pod，也可以通过`rel=beta, app=as`选中所有测试版的 Account 服务的 Pod，然后我们便可以对选中的所有 Pod 进行批量管理。

![Kubernetes 标签](/images/宏观理解Kubernetes的实现/label-2.svg "Kubernetes 标签")

一些高级对象如 ReplicaSet、Deployment 其实也是通过标签来管理的 Pod，这个我们后面会介绍到。不只是 Pod，所有对象集合的筛选基本都是基于标签匹配的。

### 第三个概念：命名空间

标签为我们提供了一个十分强大的管理方式，但是如果有多个团队在同一 K8s 集群上操作，很有可能会误操作，而且标签是很有可能会重复的，这样一个团队进行操作时很有可能会影响到另一个团队的服务，这肯定是我们不希望看到的。

为了隔绝不同团队的操作，Kubernetes 支持通过命名空间（Namespace）在一个物理集群中划分出多个相对独立的虚拟空间，这些虚拟空间就是单独的命名空间。不同的命名空间之间的资源在管理上是隔离，这样我们就可以不重不漏地将所有的对象分成若干个组，然后在某个组上再对对象进行操作。

![Kubernetes 命名空间](/images/宏观理解Kubernetes的实现/namespace.svg "Kubernetes 命名空间")

需要说明的是命名空间仅仅是对资源的管理进行了隔离，但是部署的节点并不是隔离的，不同命名空间之间的网络是否隔离也是由网络架构来决定。

在 K8s 中，Namespace 同样也是一个对象，正如前面所说的，凡是我们对 K8s 的状态进行修改，基本都是对对象的修改。

### 第四个概念：API

前面讲到，对 K8s 的操作都是通过 API，K8s 提供的 API 是基于 HTTP 的 RESTful API，对资源的查询、创建、修改、删除与 HTTP 方法GET、POST、PUT/PATCH、DELETE 对应，每个资源都有一个唯一标识的统一资源标识符，也就是请求的 PATH 中就是具体的某个资源，下面这张图举了个简单的例子：

![Kubernetes API](/images/宏观理解Kubernetes的实现/api.svg "Kubernetes API")

可以看到，每个资源的路径最开始都是以 API 组和版本开始的，然后是命名空间（集群级别的对象是没有命名空间的），然后是具体的对象类型以及对象的名称，API 的路径规则在一定程度上也反映了K8s使用命名空间隔离对象的管理的一种思想。

### 小结

我们简单介绍了 K8s 对对象的组织管理方式，简单来说，K8s 中每种对象会被至少一个 API 组来管理着，然后使用命名空间将整个集群划分成不重叠的若干个独立单元，每个单元中会有若干个对象，我们可以使用标签筛选出一个特定的对象子集，然后对这个集合的对象进行统一操作和管理，接下来我们来看几个具体的对象。

## 三、Kubernetes 上的工作负载

工作负载是用于描述业务的运行载体，是由一组 Pod 组成的抽象模型，每个 Pod 最终会产生一组应用程序容器.工作负载的作用就是自动化管理一组 Pod，利用特定的控制器来确保足够的 Pod 处于期望状态。

### 基本调度单元 Pod

Pod 是可以在 K8s 中创建和管理的最小工作单元，每个 Pod 里会包含一个或多个容器，这些容器被紧密的联系起来，运行在相同的节点上，它们的资源、存储等内容是独立的，但是网络是共享的，此外还具有很多有意思的特性，在本章会简单介绍一部分。K8s 并不允许我们对某个容器进行单独的操作，所有操作都是以 Pod 为单位的。

#### 内部结构

Pod 是 K8s 最基本的调度单元，了解 Pod 的内部结构对微服务的设计有着十分重要的意义。在一个 Pod 中，容器、卷、网络这三个是十分重要的部分，下面我们进行简单介绍：

**第一部分：容器**

每一个 Pod 的规格中都可以包含两种容器，这两种容器的职责是不同的，一种是初始化容器，叫作 InitContainer，这种容器会在 Pod 初始化时按照声明的顺序依次运行，用于初始化一些配置，运行完成之后便会退出；另一种是运行业务程序的容器，这种容器会在所有初始化容器退出之后运行，主要作用便是对外提供服务或处理特定任务。它们一般会一直存活着，使 Pod 处于 Running 状态，容器错误退出后就会自动重启，但是如果容器以 0 状态码正常退出，便不会再重启。

![Pod 多容器关系](/images/宏观理解Kubernetes的实现/pause.svg "Pod 多容器关系")

需要说明的是，在一个 Pod 中，不仅初始化容器可以存在多个，业务容器也可以存在多个，这种设计的原因是基于每个容器应当只运行一个工作进程的设计原则，否则我们需要自己来保证容器的正确运行，一个 Pod 包括多个容器正与一个应用程序存在多个进程相对应，而且分容器也可以进行更加细粒度的配置。但是如果一个应用程序存在多个进程，而且这些进程需要通过 IPC (进程间通信，Inter-Process Communication) 进行通信，那么这些进程应当运行在同一个机器上，而容器之间又是相对隔离的，没有办法让多个进程像在同一个机器上一样运行。为了解决这个问题，Pod 还引入了第三种容器—— Infrastructure Contrainer，一般被称为 pause 容器或沙箱容器，这种容器的作用是为 Pod 中其他所有的容器提供相同的网络等基础运行环境，让它们享有相同的 Linux 命名空间，即共享 PID、IPC、Network 和 UTS 等命名空间。不同容器之间也可能通过文件的方式进行共享数据，这个是通过卷（Volume）来实现的，在后面会介绍到。

**第二部分：网络**

因为 pause 容器的存在，在网络的角度看，同一个 Pod 中的不同容器就像运行在同一个专有主机上，它们享有相同的 lo 回环设备，可以通过 localhost 进行通信。由于所有容器共享相同的网络，所以不同容器不能绑定相同的端口。

![Pod 网络](/images/宏观理解Kubernetes的实现/kubenet.svg "Pod 网络")

K8s 对 Pod 间通信的网络模型并没有具体的实现，仅有一些最为基本的要求：K8s 要求每个 Pod 都有自己的 IP 地址；同时要求在不使用 NAT 技术的情况下，无论 Pod 是否运行在同一 Node 上，都可以直接通过 IP 进行访问，这就要求所有 Pod 的 IP 在集群中必须是唯一的。
K8s 把网络接口开放出来，需要在满足上述要求的前提下通过插件的方式进行实现，开源的具体实现有很多，比如 Flannel、Calico 等等。我们仅来看一个最为基本的网络方案 Kubenet 的实现方式：
Kubenet 是在每个节点上创建一个网桥 cbr0 用于连通容器网络和宿主机网络，而每个网桥都会被分配一个不重叠的网段，与这个网桥相连的所有 Pod 的网络地址都将在这个网段中，这样每个 Pod 就会有一个集群中唯一的 IP 地址。Pod 的网络接口与网桥并不是直接相连的，而是通过一个 veth 虚拟网络设备，每个 Pod 都会创建一个这种设备，它与 Pod 中的 eth0 网卡是一对，eth0 位于 Pod 的命名空间中，被所有的容器所共享，而 veth 位于宿主机的命名空间中，与网桥相连，这样便实现了容器到宿主机的网络连通。

**第三部分：卷**

再回到同一 Pod 中多容器进行通信或数据共享的问题上，很多时候我们需要使用本地文件进行共享数据，pause 容器可以帮助我们实现这个吗？答案是不行，因为每个容器的文件系统是独立的，pause 容器并没有让不同的容器共享相同的文件系统，而且不同容器本身就应当是独立的，共享相同的文件系统也不合理。

K8s 允许我们使用卷（Volume）的方式在不同容器之间共享文件，卷就是一个用于声明实际存储位置的对象，实际存储位置的类型是多种多样的，我们等会儿会举几个例子进行介绍。对于容器来说，卷是可以被挂载的，挂载就是将卷映射到容器的一个特定目录下，挂载之后，容器中的进程就可以像访问普通文件或目录一样进行访问，只不过该文件或目录的实际位置不再是容器中的读写层，而是与卷相关联的位置。卷的挂载与 Pod 具有相同的生命周期，会随着 Pod 的删除而卸载。

![Pod 存储](/images/宏观理解Kubernetes的实现/volume.svg "Pod 存储")

为了实现不同容器之间的数据共享，我们一般会使用 EmptyDir 这种类型的卷，这种卷的存储完全是临时的，可以是磁盘也可以是内存，当 Pod 被删除后，卷消失了，里面的数据也就消失了。

除了临时目录，我们在某些时刻可能还想要访问宿主机目录，比较常见的一个场景就是容器需要使用比较重的一些大数据处理类的工具包，为了减小容器的大小，这种工具包一般会被安装在宿主机上，这时候我们便可以声明 HostPath 这种卷，来达到容器访问宿主机文件的目的。

卷不仅能解决数据共享的问题还能解决持久化存储的问题，首先我们先简单介绍一下与持久存储相关的两个概念 PV（PersistentVolume，持久卷）和 PVC（PersistentVolumeClaim，持久卷申领）。PV 是集群中的一块存储，底层可以是各种各样的网络存储，比如 NFS、Ceph等。PV 与节点类似是集群资源，也就是并不在命名空间下面，PV 需要由 K8s 管理员事先进行声明。而 PVC 是用户对存储的一个申请，PVC 与 PV 的关系我们可以类比 Pod 与 Node，Pod 消耗的是 Node 的 CPU、内存资源，而 PVC 消耗的是 PV 的存储资源，但是一个 PV 只能同时被一个 PVC 绑定，所以使用 PVC 时可能会创建大量的关联 PV，所以一般会使用存储类 StorageClass 和对应的适配程序根据 PVC 所声明的规格自动创建相同大小的 PV。Pod是没有办法直接使用 PV 中的空间的，但我们可以把卷与 PVC 相关联，这样存储的位置就转移到了持久化的存储系统中。

![PV 与 PVC 的关系](/images/宏观理解Kubernetes的实现/pv-pvc.svg "PV 与 PVC 的关系")

需要说明的是刚刚讲到的与宿主机共享文件不应被视为持久化存储，因为当 Pod 由于一些原因被调度到其他节点上时就没有办法再获取到原来的数据，也就是说这个 Pod 过于依赖某个节点了，这与持久化存储是有明显的区别的。

**小结**

Pod 作为 K8s 的核心，其设计是十分巧妙的，我们仅介绍了 Pod 的三大核心部分——容器、网络、卷，下面这张图基本把刚刚讲的内容全部涵盖了，相信能带来一个比较全面且直观的理解。我们在下一节生命周期中还会讲到一些内容，Pod 详细的一些规格定义不作过多介绍，可以到官方 API 文档中查看 [Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#podspec-v1-core)。

![Pod 核心组成](/images/宏观理解Kubernetes的实现/pod-global.svg "Pod 核心组成")

#### 生命周期

理解 Pod 的运作必然要了解 Pod 的整个生命周期，首先我们需要先了解三个概念：

**第一个概念：回调 （Hooks）**

我们之前说过，一个容器的业务工作进程应当只有一个，但这个进程很有可能依赖其他的服务，比如 Nginx ，我们希望当容器启动时就能自动启动 Nginx ，而不是进入容器中手动启动，同样在容器被销毁时，我们也希望能够主动关闭这些进程，以保证服务的正确退出。为了实现这种目的，K8s 支持我们为每个容器设置两种回调，一种是 postStart，这种回调会在容器启动之后立即执行；另一种是 preStop，这种回调会在接收到 kill 信号时执行，这两种回调都支持执行容器中的命令或向容器发送 HTTP GET 请求两种方式。

![Pod Hooks](/images/宏观理解Kubernetes的实现/hook.svg "Pod Hooks")

需要注意的是，如果 postStart 没有正确被执行或返回了退出状态，容器就会被重启。

对于 Pod 的终止过程，我们再多介绍一些，当容器确定要被杀死之后，如果我们设置了 preStop，那么会立即执行 preStop 中的命令，执行完成之后，无论成功与否，K8s 都会向容器中的业务进程发送 TERM 信号，如果从确定要被杀死开始经过了一段时间，进程还在运行着，K8s 会发送强制终止 KILL 信号，这个时间有两种，一种叫做 `terminationGracePeriodSeconds`，用于容器出现问题自动重启的情况，另一种叫 `deletionGracePeriodSeconds`，用于我们主动删除 Pod，这个参数在执行删除命令的时候可以设置的。

![Pod 终止过程](/images/宏观理解Kubernetes的实现/terminal.svg "Pod 终止过程")

**第二个概念：探针（Probe）**

K8s 是一个容器自动化管理工具，而容器出现问题异常退出是经常发生的，所以对容器进行健康检查确保容器正常运行是一项特别重要的工作。K8s 实现这一功能的方案叫做探针（Probe）。探针可以是一条可执行的命令，也可以是 HTTP GET 请求，还可以是一个 TCP Socket 链接，当探针返回正常的状态或者能够成功建立连接时将被视为成功，否则被视为失败。失败之后 K8s 会根据重启策略重启容器，探针会隔一段时间执行一次，以确保容器处于正常的状态。探针根据功能分为三种—— liveness probe、startup probe、readiness probe，在介绍这三个探针之前，我们需要先了解一下容器的重启策略。

与预期可能不一样的是，容器的重启策略并不是针对某个容器进行独立设置，而是针对 Pod 进行设置，这个策略对于该 Pod 中的所有容器都会生效，主要有三种策略：Always、OnFailure 和 Never，这三种策略的含义还是比较清晰的，我们不作过多介绍，以 OnFailure 为例，如果容器的退出状态不是0，那么 K8s 会按照指数回退方式计算重启延迟（10s、20s、40s、...）,这个延迟最长为 5 分钟，当容器正常运行 10 分钟时，该延迟会进行重置。

![探针 Probe](/images/宏观理解Kubernetes的实现/probe.svg "探针 Probe")

liveness 探针用来指示容器是否正在运行，如果该探针失败了，那么该容器会被杀死，然后根据重启策略进行重启，为了防止因为一次异常失败就将容器重启带来大的启动开销，探针是可以设置失败阈值（failureThreshold）的，只有连续失败次数达到了这个阈值才被判定为探测失败。仅有失败阈值是不够的，因为一个进程的启动时需要时间的，如果容器一开始运行就进行探测，那么大概率是会失败的，所以我们还可以通过设置延迟时间（initialDelaySeconds）来延迟探测。
但是对于很多进程来说，启动的时间每次可能差别很大，仅仅用 initialDelaySeconds 也存在问题，所以 K8s 提供了一个叫做 startupProbe 的启动探针，如果设置了这个探针，在容器启动时会只执行该探针，当这个探针成功之后才会执行其他的探针。为了解决刚刚的问题，我们可以设置一个启动探针，把这个探针的失败阈值设置的比较高，同时把探测的时间间隔也设置的大一些，这样就能防止服务没有启动就被重启，与 liveness 探针类似，如果启动探针失败了，K8s 也会根据重启策略重启容器。

似乎拥有了以上两种探针就能够探测容器是否健康了，但是事实上，正在运行的容器也可能无法正常提供服务，这时候我们需要把这个 Pod 的 IP 从服务列表中删除，防止访问到这个不可用的 Pod。所以就又有了一个叫做 readinessProbe 的探针，这个探针用来指示容器是否准备好为请求提供服务，参数与 liveness 探针相同，不同的是失败后并不会重启容器，而是从服务列表中删除该 Pod 的 IP。

**第三个概念：状态（Status）**

Pod 所处的阶段（Parse）是由它所包括的所有容器的状态来决定的，如果容器未启动 Pod 就会处于 Pending 状态，如果存在至少一个容器在运行着，那么 Pod 会处于 Running 状态，当所有的容器都正常退出了，Pod 会处于 Succeeded 状态，否则处于 Failed 状态，如果 K8s 与 Pod 的通信出现了问题，那么这个状态会变为 Unknown。

![Pod Status](/images/宏观理解Kubernetes的实现/pod-status.svg "Pod Status")

上面说的这几个状态只是简单地说明了 Pod 所处的阶段（Parse），除此之外，每个 Pod 还包含一组条件（Conditions），K8s 会对这些条件进行探测（包括前面说到的探针），探测成功则为 True，否则为 False，这些 Conditions 的组合才是 K8s 进行管理的依据。
Pod 默认有以下四个 Condition：
- `PodScheduled`：Pod 已经被调度到某节点；
- `ContainersReady`：Pod 中所有容器都已就绪；
- `Initialized`：所有的 Init 容器都已成功运行完；
- `Ready`：Pod 可以为请求提供服务，并且应该被添加到对应服务的 IP 列表中。

除了这四个，我们也可以自己定义一些 Condition， 如下所示：

![自定义 Condition 与 ReadinessGates](/images/宏观理解Kubernetes的实现/readiness-gate.svg "自定义 Condition 与 ReadinessGates")

我们的应用可以反馈我们自己定义的 Condition 的状态，解释一下为什么要自定义 Condition。在一些时候，K8s 探测到的状态可能不能满足我们的需求，有时候所有 Container 都已经 Ready 了（使用 readiness 探针进行探测），但是可能这个 Pod 因为其他原因并不应该被能够被访问，这时候我们就希望有一个字段能够来控制这个 Pod 是否 Ready，而不仅仅是依赖于所有容器是否 Ready，我们就可以定义一个 readinessGates，这里面包含了我们定义的一些 Condition，这些 Condition 可以由我们的应用来进行反馈，只有 ContainersReady 和 readinessGates 中的所有 Condition 都为 True 时，这个 Pod 才被视为 Ready 且可以提供服务。

**一个 Pod 的完整生命周期**

下面这张图较为完整的展示了 Pod 的生命周期，首先 Scheduler 会对 Pod 进行调度，把这个 Pod 调度到一个 Node 上，然后会启动 pause 容器来为其他容器提供基础命名空间，之后会创建卷，然后依次逐个启动 init 容器，之后启动业务容器，并使用定义的探针进行探测状态，当探测失败时会根据重启策略重启该容器，删除 Pod 时会先杀死 pause 容器，再杀死业务容器。

![Pod 生命周期](/images/宏观理解Kubernetes的实现/pod-lifecycle.svg "Pod 生命周期")

### 自动管理 Pod 的资源 ReplicaSet & Deployment

Pod 作为最基本的运行单元，本身是没有办法自动进行管理保证正确运行的，这就需要一些更高级的对象来完成这个任务，K8s 提供了多种工作负载资源：在集群中运行多个副本的 ReplicaSet、每个节点上运行一个 Pod 的 DaemonSet、保证运行 N 次的 Job、定时执行任务的 Cronjob 、可以进行滚动更新的 Deployment、有状态的 StatefulSet 等等。

我们仅简单介绍 ReplicaSet 和 Deployment 这两个比较重要的对象。

ReplicaSet 可以在集群中保证运行一定副本数量的 Pod ，这些 Pod 的规格都是一样的，他们根据 ReaplicaSet 中的模板生成，当某个 Pod 由于一些原因被销毁时 ReplicaSet 会自动创建新的 Pod，以维持总的数量与所设定的一致。

![ReplicaSet](/images/宏观理解Kubernetes的实现/replicaset.svg "ReplicaSet")

ReplicaSet 判断 Pod 是否应该被自己管理并不是依据 Pod 的规格与模板是否一致，而是仅仅通过标签来决定，也就是说如果我们把一个无关 Pod 的标签修改一下，让 ReplicaSet 匹配上，那么 ReplicaSet 也会认为这个 Pod 应该被自己管理。那么如果我们想要对服务进行升级，仅仅修改模板是没有作用的，因为要匹配的标签是没有变化的，如果我们改变了这个标签，之前创建的 Pod 就不被 ReplicaSet 管理了，并不会自动删除，也达不到升级服务的目的。

为了解决更新的问题，K8s 为我们提供了更高级的对象——Deployment，这个对象的规格与 ReplicaSet 类似，都包含着 Pod 的模板，它并不是直接创建 Pod ，而是根据 Pod 模板创建 ReplicaSet ，然后 ReplicaSet 再创建 Pod，这样当我们需要对服务进行升级时，只要改一下 Deployment 的 Spec 里面的 Pod 模板，这样由于 Pod 模板变了，就会再创建一个新的 ReplicaSet，同时将原有的 ReplicaSet 的副本数设置为 0，这样 K8s 中就只有最新版本的 Pod 了。

![Deployment](/images/宏观理解Kubernetes的实现/deployment.svg "Deployment")

Deployment 能做的远不止这样，它还能做到零停机、不停服的滚动更新，也就是所有的 Pod 一部分一部分得用新版本进行替换掉，以此来保证服务不中断。接下来我们介绍两个重要参数，MaxSurge 和 MaxUnavailable，这两个参数可以是整数，也可以是相对于所有实例数的百分比。MaxSurge 实际上是更新过程中所有服务副本的上限，MaxUnavailable 是更新过程中可用服务副本（处于 Ready 状态的 Pod）数量的下限。以 replica=3，maxSurge=20%，maxUnavailable=20% 为例，在更新过程中，所有副本数量不会超过 3+⌈3\*20%⌉=4，可用副本数量不会低于 3-⌈3\*20%⌉=2，K8s 会在这两个参数允许的范围内以尽可能大的比例进行升级，如下图所示：

![滚动更新](/images/宏观理解Kubernetes的实现/rolling-update.svg "滚动更新")

至于 DaemonSet、Cronjob、Job 这些对象，我们不再介绍，他们根本上都是去确保符合一定条件的可用 Pod 与预期一致。除了这些工作负载，我们再简单介绍一下另一个重要的对象 Service。

### 提供稳定访问能力的资源 Service

我们使用工作负载为某个服务创建了若干个 Pod 之后，尽管我们可以通过 Pod 的 IP 和端口在 K8s 内访问服务，但是 Pod 是一个具有短暂生命周期的对象，如果我们的 Pod 因为升级或节点故障等原因重建了，那么这个 Pod 的 IP 和监听端口也就变了，我们没有办法使用原有的 IP 和端口继续访问该服务，除此以外，一个服务的副本数可能有多个，我们还需要对所有 Pod 的访问进行负载均衡，所以我们需要一个在这些 Pod 之上的、不容易发生变化的抽象对象，在 K8s 中，这个抽象层叫做 Service。

![Service](/images/宏观理解Kubernetes的实现/service.svg "Service")

Service 具有固定的 IP 和端口，Service 实际上是对一组 IP 进行负载均衡，这组 IP 被放到一个叫做 Endpoints 的对象中，Service 控制器会通过标签筛选出一组 Pod，然后将这组 Pod 的 IP 和端口自动维护到 Endpoints 中， 当访问这个服务时，会使用特定的负载均衡算法取出某个 IP 来进行访问。

当一个 Service 被创建之后，K8s 会立即为这个 Service 分配一个虚拟 IP 地址，之后会将这个服务的信息通知给所有节点的 kube-proxy 组件，让该服务可以在所有节点上进行寻址，我们以现在最常用的 iptables 模式为例，当要访问一个 Service 时，节点上的内核会根据 iptables 规则将这个报文的目的地址使用这个 Service 对应的 Endpoints 中一个 IP 地址进行替换，以此来访问具体的服务实例。

![访问服务](/images/宏观理解Kubernetes的实现/service-request.svg "访问服务")

### 小结

我们仅简单介绍了 K8s 比较核心的几个资源，除了我们介绍的这些，K8s 还支持 DaemonSet、Cronjob、Job、StatefulSet、ConfigMap、Secret、ResourceQuota 这些高级资源，甚至我们可以自己定义一个资源，可扩展性是 K8s 得以广泛推广使用的一个重要原因。

## 四、Kubernetes 的架构

### 第一部分：核心组件

![Kubernetes 架构](/images/宏观理解Kubernetes的实现/arch.svg "Kubernetes 架构")

K8s 集群由一组节点组成，每个节点是一个虚拟机或物理机，这些节点被分成两类，其中一类是控制平面节点，或被称为主节点，这类节点一般运行着相同的控制组件，而且不会运行用户创建的容器，我们先来看一下主节点里的组件：
- `etcd`：兼具一致性和高可用性的键值数据库，作为保存 Kubernetes 所有集群数据的后台数据库
- `API server`：它是 K8s 所有操作必须经过的一个节点，也是唯一一个能操作 etcd 的组件，其他所有组件都与 API server 进行通信
- `Scheduler`：负责调度，是 K8s 进行资源调度的地方，为新创建的、未指定运行节点（node）的  Pods 根据调度策略分配节点
- `Controller Manager`：是一系列资源控制器 Controller 的集合，每个资源都会有自己的 Controller，负责监听相应对象的状态并进行创建、修改、删除等操作，如 Node 、ReplicaSet、Deployment 都有相对应的 Controller

除了控制面节点，还有一类节点被称为工作节点，这类节点是运行用户 Pod 的地方，这类节点一般会包括三个组件：
- `Kubelet`：从 API server 获取所处节点应运行的 Pod 的规格，确保容器健康运行，同时监听所处节点所有 Pod 的状态并通知给 API server
- `Kube proxy`：每个节点上的网络代理，维护节点上的网络规则，用于实现 Service 对象
- `Container Runtime`：负责运行容器的软件，除 Docker 外， 还可以是 containerd 等


### 第二部分：组件通信

API server 是唯一一个能与 etcd 通信并进行查询、修改集群状态等操作的组件，其他组件都需要通过 API server 来获取所需要的信息，K8s 使用了一种 List-Watch 的机制来实现组件与 API server 的通信，下面这张图展示了通信的主要流程：当组件与 API server 建立链接之后首先会获取到所有的 Pod 列表，如果我们创建了一个 Pod，首先 API server 会将这个 Pod 的信息写入 etcd，写成功之后会再向所有监听 Pod 的客户端发送新建或更新的 Pod 的信息，为了防止漏掉消息，组件每隔一段时间还会 relist 重新获取所有的 Pod 列表。

![List-Watch](/images/宏观理解Kubernetes的实现/list-watch.svg "List-Watch")

### 第三部分：提高集群可用性

为了提高集群的可用性，可以存在多个 master，每个 master 可以运行所有需要的组件，每个 master 的 API server 只与自己节点上的 etcd 进行通信，所以 etcd 之间只需要保证数据一致，而不需要进行负载均衡，而 API servers 之间是需要进行负载均衡的。但是 Scheduler 与 Controller Manager 只会有一个副本处于活动状态，其余的都会处于等待状态，他们等着活动状态的主节点出现故障才会竞争变更到活动状态，如下图所示：

![高可用架构](/images/宏观理解Kubernetes的实现/ha.svg "高可用架构")

## 五、Kubernetes 的调度流程

### 理解资源分配

K8s 的核心就是调度，首先我们先看一下 K8s 对 CPU、内存这种资源是如何处理的。在描述 Pod 的规格时，我们可以声明每个容器所需的 CPU、内存，也就是 resources 中的 request 字段，可能会感到奇怪的一点是 K8s 并不是把 request 个物理 CPU 分配给某个容器专用，而是分配 CPU 时间，所以 K8s 所支持的最小的 CPU 单位不是核，而是千分之一核。如果 K8s 为某个容器分配了 100m 的 CPU，那么在一定时间内这个容器在所有物理 CPU 上运行的时间之和为一核 CPU 时间的十分之一，也就是说容器实际使用的物理 CPU 核数是有可能超过我们所设的数目的，所以在容器中能看到的是宿主机的核数，而不是我们分配的核数，这个如果不注意，一些根据 CPU 核数创建线程的程序可能性能受到比较大的影响。 resources 中还可以设置 limit，如下图所示：

![资源限制](/images/宏观理解Kubernetes的实现/resource.svg "资源限制")

**CPU**

为了理解 request、limit 和实际 CPU 使用的关系，我们来看下面这个例子：

现在有一个 Node，可用 CPU 为1核，这个 Node 上有两个 Pod，a 和 b，a 的 request 为 100m，b 的 request 为 500m，两个 Pod 都没有设置 limit。如果两个 Pod 都尽可能得去消耗 CPU，K8s 会将所有可用 CPU 资源按照 request 比例进行分配，也就是 a 实际能得到 167m 的 CPU 资源，b 能得到 833m 的资源，实际使用的 CPU 比例与 request 的比例是相同的，也就是说，如果在所有 Pod 都去尽可能争夺CPU的时候，按这种方法分配我们是可以保证每个 Pod 都能至少有自己所 request 的 CPU 资源来运行程序，request 是一个能确保得到的资源值，所以所有容器的资源 request 之合不会超过宿主机的实际资源量。

如果 Pod b 仅仅使用了 200m 的资源，Pod a 还是尽可能去消耗 CPU，那么剩下的 800m 的 CPU 资源都会分给 Pod a，也就是会尽可能利用起 CPU 时间。

如果我们给 Pod a 的 CPU 设置一个 limit=300m，在刚刚的情况下，Pod a 所消耗的 CPU 资源不会超过 300m，尽管还有 500m 的资源空闲，所以说 limit 是一个不会超过的上限。

![CPU 调度](/images/宏观理解Kubernetes的实现/cpu.svg "CPU 调度")

**内存**

与 CPU 不同，内存是一个不具备弹性的资源，当一块内存被分配之后，除非进程退出了，否则不应被其他进程占用，所以如果所有容器申请的内存超过了宿主机所能提供的数量，那么 K8s 会按照一定的规则杀死容器，所以一般我们会将内存的 request 与 limit 设置相同，防止某个容器内存使用量没有到达 limit 却被杀死。

request 是一个确保能得到的值，所以一个 Node 上所有容器的 request 之和不会超过这个 Node 所能提供的资源量，而 limit 却是可以超过的，如下图所示：

![Request 和 Limit](/images/宏观理解Kubernetes的实现/request-limit.svg "Request 和 Limit")

所以当 K8s 的 Scheduler 为 Pod 分配 Node 时，看的是 request 而不是 limit，除了资源， nodeSelector、nodeAffinity 两个字段对 K8s 的调度也有着很大的作用，nodeSelector 可以根据标签来筛选可用节点，nodeAffinity 可以进行更宽松的偏好型的匹配，具体可以 [查看官方文档](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/)。

### Deployment 是怎么产生的 Pod

![从 Deployment 到 Pod](/images/宏观理解Kubernetes的实现/deployment-pod.svg "从 Deployment 到 Pod")

上面这张图描述了从我们创建 Deployment 到实际运行容器的整个过程，我们可以看到所有的操作都会过 API server，API server 是唯一一个能操作 etcd 的组件，整个流程主要分成以下步骤：
1. 使用 kubectl 创建一个 Deployment，API server 将这个 Deployment 的信息存入 etcd；
2. Deployment Controller 通过 Watch 机制得知创建了一个新的 Deployment，然后根据 Deployment 中的 Pod 模板创建一个新的 ReplicaSet，通过 API server 将这个 ReplicaSet 的信息存入 etcd；
3. ReplicaSet Controller 通过 Watch 机制得知创建了一个新的 ReplicaSet ，然后根据 ReplicaSet 中的 Pod 模板创建 replica 个 Pod，通过 API server 将这些 Pod 的信息存入 etcd；
4. Schedule 通过 Watch 机制得知有新的 Pod 被创建了还没有被调度，根据调度算法将 Pod 分配到一个 Node 上，并通过 API server 更新 Pod 的信息；
5. Node X 上的 kubelet 通过 Watch 机制得知被分配了一个新的 Pod ，然后通过 Docker 运行指定的容器，并将状态反馈给 API server。

之后便是 Pod 的整个生命周期的过程，在之前已经讲过了。

## 未完待续

<!-- ## 六、高级 Kubernetes

### 用户自定义资源 CRD 与控制器模式

### Webhook 模式

### 监听组件 Informer


## 七、扩展 Kubernetes

### 应用管理 Helm

### 集群联邦 KubeFed

### 层级命名空间 HNC

### 网络方案 Calico

## 八、总结 -->

## 参考

- [Marko Luksa - Kubernetes in Action](https://livebook.manning.com/book/kubernetes-in-action/kubernetes-in-action/)
- [Kubernetes - 面向信仰编程](https://draveness.me/tags/kubernetes)
- [Kubernetes 官方文档](https://kubernetes.io/zh/docs/home/)
