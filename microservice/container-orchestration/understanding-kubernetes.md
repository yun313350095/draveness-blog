---
layout: post
title: 谈 Kubernetes 的架构设计与实现原理
permalink: /understanding-kubernetes
tags: kubernetes Docker 微服务 容器编排
toc: true
desc: Kubernetes 基本上是这两年最热门、最被人熟知的技术了，它为软件工程师提供了强大的容器编排能力，模糊了开发和运维之间的边界，让我们开发、管理和维护一个大型的分布式系统和项目变得更加容易。这篇文章是整个 Kuberentes 架构设计与实现原理的开篇，文章会先简单介绍 Kuberentes 的背景、依赖的技术，它的架构以及设计理念，最后会提及一些关键概念和实现原理。
---

# 谈 Kubernetes 的架构设计与实现原理

Kubernetes 基本上是这两年最热门、最被人熟知的技术了，它为软件工程师提供了强大的容器编排能力，模糊了开发和运维之间的边界，让我们开发、管理和维护一个大型的分布式系统和项目变得更加容易。

![kubernetes-banner](https://img.draveness.me/2018-11-25-kubernetes-banner.png)

这篇文章是整个 Kuberentes 架构设计与实现原理的开篇，文章会先简单介绍 Kuberentes 的背景、依赖的技术，它的架构以及设计理念，最后会提及一些关键概念和实现原理。

## 介绍

作为一个目前在生产环境已经广泛使用的开源项目 Kubernetes 被定义成一个用于自动化部署、扩容和管理容器应用的开源系统；它将一个分布式软件的一组容器打包成一个个更容易管理和发现的逻辑单元。

Kubernetes 是希腊语『舵手』的意思，它最开始由 Google 的几位软件工程师创立，深受公司内部 [Borg](https://ai.google/research/pubs/pub43438) 和 Omega 项目的影响，很多设计都是从 Borg 中借鉴的，同时也对 Borg 的缺陷进行了改进，Kubernetes 目前是 Cloud Native Computing Foundation (CNCF) 的项目并且是很多公司管理分布式系统的解决方案。

![docker-swarm-vs-kubernetes](https://img.draveness.me/2018-11-25-docker-swarm-vs-kubernetes.png)

在 Kubernetes 统治了容器编排这一领域之前，其实也有很多容器编排方案，例如 [compose](https://github.com/docker/compose) 和 [Swarm](https://github.com/docker/swarm)，但是在运维大规模、复杂的集群时，这些方案基本已经都被 Kubernetes 替代了。

Kubernetes 将已经打包好的应用镜像进行编排，所以如果没有容器技术的发展和微服务架构中复杂的应用关系，其实也很难找到合适的应用场景去使用，所以在这里我们会简单介绍 Kubernetes 的两大『依赖』——容器技术和微服务架构。

### 容器技术

Docker 已经是容器技术的事实标准了，作者在前面的文章中 [Docker 核心技术与实现原理](https://draveness.me/docker) 曾经介绍过 Docker 的实现主要依赖于 Linux 的 namespace、cgroups 和 UnionFS。

![docker-logo](https://img.draveness.me/2018-11-25-docker-logo.png)

它让开发者将自己的应用以及依赖打包到一个可移植的容器中，让应用程序的运行可以实现环境无关。

我们能够通过 Docker 实现进程、网络以及挂载点和文件系统隔离的环境，并且能够对宿主机的资源进行分配，这能够让我们在同一个机器上运行多个不同的 Docker 容器，任意一个 Docker 的进程都不需要关心宿主机的依赖，都各自在镜像构建时完成依赖的安装和编译等工作，这也是为什么 Docker 是 Kubernetes 项目的一个重要依赖。

### 微服务架构

如果今天的软件并不是特别复杂并且需要承载的峰值流量不是特别多，那么后端项目的部署其实也只需要在虚拟机上安装一些简单的依赖，将需要部署的项目编译后运行就可以了。

![microservices](https://img.draveness.me/2018-11-25-microservices.png)

但是随着软件变得越来越复杂，一个完整的后端服务不再是单体服务，而是由多个职责和功能不同的服务组成，服务之间复杂的拓扑关系以及单机已经无法满足的性能需求使得软件的部署和运维工作变得非常复杂，这也就使得部署和运维大型集群变成了非常迫切的需求。

### 小结

Kubernetes 的出现不仅主宰了容器编排的市场，更改变了过去的运维方式，不仅将开发与运维之间边界变得更加模糊，而且让 DevOps 这一角色变得更加清晰，每一个软件工程师都可以通过 Kubernetes 来定义服务之间的拓扑关系、线上的节点个数、资源使用量并且能够快速实现水平扩容、蓝绿部署等在过去复杂的运维操作。

## 设计

这一小节我们将介绍 Kubernetes 的一些设计理念，这些关键字能够帮助了解 Kubernetes 在设计时所做的一些选择：

![kubernetes-design](https://img.draveness.me/2018-11-25-kubernetes-design.png)

这里将按照顺序分别介绍声明式、显式接口、无侵入性和可移植性这几个设计的选择能够为我们带来什么。

### 声明式

声明式（Declarative）的编程方式一直都会被工程师们拿来与命令式（Imperative）进行对比，这两者是完全不同的编程方法。我们最常接触的其实是命令式编程，它要求我们描述为了达到某一个效果或者目标所需要完成的指令，常见的编程语言 Go、Ruby、C++ 其实都为开发者了命令式的编程方法，

在 Kubernetes 中，我们可以直接使用 YAML 文件定义服务的拓扑结构和状态：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rss-site
  labels:
    app: web
spec:
  containers:
    - name: front-end
      image: nginx
      ports:
        - containerPort: 80
    - name: rss-reader
      image: nickchase/rss-php-nginx:v1
      ports:
        - containerPort: 88
```

这种声明式的方式能够大量地减少使用者的工作量，极大地增加开发的效率，这是因为声明式能够简化需要的代码，减少开发人员的工作，如果我们使用命令式的方式进行开发，虽然在配置上比较灵活，但是带来了更多的工作。

```sql
SELECT * FROM posts WHERE user_id = 1 AND title LIKE 'hello%';
```

SQL 其实就是一种常见的声明式『编程语言』，它能够让开发者自己去指定想要的数据是什么，Kubernetes 中的 YAML 文件也有着相同的原理，我们可以告诉 Kubernetes 想要的最终状态是什么，而它会帮助我们从现有的状态进行迁移。

![kubernetes-declarative-api](https://img.draveness.me/2018-11-25-kubernetes-declarative-api.png)

如果 Kubernetes 采用命令式编程的方式提供接口，那么工程师可能就需要通过代码告诉 Kubernetes 要达到某个状态需要通过哪些操作，相比于更关注状态和结果声明式的编程方式，命令式的编程方式更强调过程。

总而言之，Kubernetes 中声明式的 API 其实指定的是集群期望的运行状态，所以在出现任何不一致问题时，它本身都可以通过指定的 YAML 文件对线上集群进行状态的迁移，就像一个水平触发的系统，哪怕系统错过了相应的事件，最终也会根据当前的状态自动做出做合适的操作。

### 显式接口

第二个 Kubernetes 的设计规范其实就是 —— 不存在内部的私有接口，所有的接口都是显示定义的，组件之间通信使用的接口对于使用者来说都是显式的，我们都可以直接调用。

![kubernetes-external-api](https://img.draveness.me/2018-11-25-kubernetes-external-api.png)

当 Kubernetes 的接口不能满足工程师的复杂需求时，我们需要利用已有的接口实现更复杂的特性，在这时 Kubernetes 的这一设计就不会成为自定义需求的障碍。

### 无侵入性

为了尽可能满足用户（工程师）的需求，减少工程师的工作量与任务并增强灵活性，Kubernetes 为工程师提供了无侵入式的接入方式，每一个应用或者服务一旦被打包成了镜像就可以直接在 Kubernetes 中无缝使用，不需要修改应用程序中的任何代码。

![kuberentes-non-invasive](https://img.draveness.me/2018-11-25-kuberentes-non-invasive.png)

Docker 和 Kubernetes 就像包裹在应用程序上的两层，它们两个为应用程序提供了容器化以及编排的能力，在应用程序内部却不需要任何的修改就能够在 Docker 和 Kubernetes 集群中运行，这是 Kubernetes 在设计时选择无侵入带来最大的好处，同时无侵入的接入方式也是目前几乎所有应用程序或者服务都必须考虑的一点。

### 可移植性

在微服务架构中，我们往往都会让所有处理业务的服务变成无状态的服务，以前在内存中存储的数据、Session 等缓存，现在都会放到 Redis、ETCD 等数据库中存储，微服务架构要求我们对业务进行拆分并划清服务之间的边界，所以有状态的服务往往会对架构的水平迁移带来障碍。

然而有状态的服务其实是无可避免的，我们将每一个基础服务或者业务服务都变成了一个个只负责计算的进程，但是仍然需要有其他的进程负责存储易失的缓存和持久的数据，Kubernetes 对这种有状态的服务也提供了比较好的支持。

Kubernetes 引入了 `PersistentVolume` 和 `PersistentVolumeClaim` 的概念用来屏蔽底层存储的差异性，目前的 Kubernetes 支持下列类型的 `PersistentVolume`：

![kubernetes-persistent-volume](https://img.draveness.me/2018-11-25-kubernetes-persistent-volume.png)

这些不同的 `PersistentVolume` 会被开发者声明的 `PersistentVolumeClaim` 分配到不同的服务中，对于上层来讲所有的服务都不需要接触 `PersistentVolume`，只需要直接使用 `PersistentVolumeClaim` 得到的卷就可以了。

## 架构

Kubernetes 遵循非常传统的客户端服务端架构，客户端通过 RESTful 接口或者直接使用 kubectl 与 Kubernetes 集群进行通信，这两者在实际上并没有太多的区别，后者也只是对 Kubernetes 提供的 RESTful API 进行封装并提供出来。

![kubernetes-architecture](https://img.draveness.me/2018-11-25-kubernetes-architecture.png)

每一个 Kubernetes 就集群都由一组 Master 节点和一系列的 Worker 节点组成，其中 Master 节点主要负责存储集群的状态并为 Kubernetes 对象分配和调度资源。

### Master

作为管理集群状态的 Master 节点，它主要负责接收客户端的请求，安排容器的执行并且运行控制循环，将集群的状态向目标状态进行迁移，Master 节点内部由三个组件构成：

![kubernetes-master-node](https://img.draveness.me/2018-11-25-kubernetes-master-node.png)

其中 API Server 负责处理来自用户的请求，其主要作用就是对外提供 RESTful 的接口，包括用于查看集群状态的读请求以及改变集群状态的写请求，也是唯一一个与 etcd 集群通信的组件。

而 Controller 管理器运行了一系列的控制器进程，这些进程会按照用户的期望状态在后台不断地调节整个集群中的对象，当服务的状态发生了改变，控制器就会发现这个改变并且开始向目标状态迁移。

最后的 Scheduler 调度器其实为 Kubernetes 中运行的 Pod 选择部署的 Worker 节点，它会根据用户的需要选择最能满足请求的节点来运行 Pod，它会在每次需要调度 Pod 时执行。

### Worker

其他的 Worker 节点实现就相对比较简单了，它主要由 kubelet 和 kube-proxy 两部分组成：

![kubernetes-worker-node](https://img.draveness.me/2018-11-25-kubernetes-worker-node.png)

kubelet 是一个节点上的主要服务，它周期性地从 API Server 接受新的或者修改的 Pod 规范并且保证节点上的 Pod 和其中容器的正常运行，还会保证节点会向目标状态迁移，该节点仍然会向 Master 节点发送宿主机的健康状况。

另一个运行在各个节点上的代理服务 kube-proxy 负责宿主机的子网管理，同时也能将服务暴露给外部，其原理就是在多个隔离的网络中把请求转发给正确的 Pod 或者容器。

## 实现原理

到现在，我们已经对 Kubernetes 有了一些简单的认识和了解，也大概清楚了 Kubernetes 的架构，在这一小节中我们将介绍 Kubernetes 中的一些重要概念和实现原理。

### 对象

Kubernetes 对象是系统中的持久实体，它使用这些对象来表示集群中的状态，这些对象能够描述：

![kubernetes-object](https://img.draveness.me/2018-11-25-kubernetes-object.png)

这些对象描述了哪些应用应该运行在集群中，它们请求的资源下限和上限以及重启、升级和容错的策略。每一个创建的对象其实都是我们对集群状态的改变，这些对象描述的其实就是集群的期望状态，Kubernetes 会根据我们指定的期望状态不断检查对当前的集群状态进行迁移。

```go
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

每一个对象都包含两个嵌套对象来描述规格（Spec）和状态（Status），对象的规格其实就是我们期望的目标状态，而状态描述了对象的当前状态，这部分一般由 Kubernetes 系统本身提供和管理，是我们观察集群本身的一个接口。

### Pod

Pod 是 Kubernetes 中最基本的概念，它也是 Kubernetes 对象模型中我们可以创建或者部署的最小并且最简单的单元。

![kubernetes-pod](https://img.draveness.me/2018-11-25-kubernetes-pod.png)

它将应用的容器、存储资源以及独立的网络 IP 地址等资源打包到了一起，表示一个最小的部署单元，但是每一个 Pod 中的运行的容器可能不止一个，这是因为 Pod 最开始设计时就能够在多个进程之间进行协调，构建一个高内聚的服务单元，这些容器能够共享存储和网络，非常方便地进行通信。

### 控制器

最后要介绍的就是 Kubernetes 中的控制器，它们其实是用于创建和管理 Pod 的实例，能够在集群的层级提供复制、发布以及健康检查的功能，这些控制器其实都运行在 Kubernetes 集群的主节点上。

在 Kuberentes 的 [kubernetes/pkg/controller/](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller) 目录中包含了官方提供的一些常见控制器，我们可以通过下面这个函数看到所有需要运行的控制器：

```go
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	// ...

	return controllers
}
```

这些控制器会随着控制器管理器的启动而运行，它们会监听集群状态的变更来调整集群中的 Kuberentes 对象的状态，在后面的文章中我们会展开介绍一些常见控制器的实现原理。

## 总结

作为 Kubernetes 系列文章的开篇，我们已经了解了它出现的背景、依赖的关键技术，同时我们也介绍了 Kubernetes 的架构设计，主节点负责处理客户端的请求、节点的调度，最后我们提到了几个 Kuberentes 中非常重要的概念：对象、Pod 和控制器，在接下来的文章中我们会深入介绍 Kuberentes 的实现原理。

## Reference

+ [Large-scale cluster management at Google with Borg](https://ai.google/research/pubs/pub43438)
+ [Production-Grade Container Orchestration](https://kubernetes.io/)
+ [Borg, Omega, and Kubernetespo](https://ai.google/research/pubs/pub44843)
+ [Kubernetes Design and Development Explained](https://thenewstack.io/kubernetes-design-and-development-explained/)
+ [2018 is the year of Kubernetes – with some help from Docker](https://www.dynatrace.com/news/blog/2018-year-of-kubernetes-with-help-from-docker/?utm_source=tuicool&utm_medium=referral)
+ [Introduction to Kubernetes Architecture](https://x-team.com/blog/introduction-kubernetes-architecture/)
+ [Setting up highly available Kubernetes clusters](https://elastisys.com/wp-content/uploads/2018/01/kubernetes-ha-setup.pdf?x83281)
+ [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
