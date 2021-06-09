---
title: K8s Operator之controller-runtime
tags:
  - K8s
  - golang
  - operator
abbrlink: b61a3aa4
date: 2021-06-01 16:58:39
---

# k8s Operator

## 简介

官方介绍：[kubernetes/operator](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/)

Kubernetes Operator是一种封装、部署和管理 Kubernetes 应用的方法。我们使用 Kubernetes API（应用编程接口）和 kubectl 工具在 Kubernetes 上部署并管理 Kubernetes 应用。也是一种特定于应用的控制器，可扩展 Kubernetes API 的功能，来代表 Kubernetes 用户创建、配置和管理复杂应用的实例。它基于基本 Kubernetes 资源和控制器概念构建，但又涵盖了特定于域或应用的知识，用于实现其所管理软件的整个生命周期的自动化。 

## 工作原理

在 Kubernetes 中，控制平面的控制器实施**控制循环**，反复比较集群的理想状态和实际状态。如果集群的实际状态与理想状态不符，控制器将采取措施解决此问题。 

Operator 是使用自定义资源（Custom Resource, aka. CR）管理应用及其组件的自定义 Kubernetes 控制器。高级配置和设置由用户在 CR 中提供。Kubernetes Operator 基于嵌入在 Operator 逻辑中的最佳实践将高级指令转换为低级操作。

自定义资源是 Kubernetes 中的 API 扩展机制。自定义资源定义（CRD）会明确 CR 并列出 Operator 用户可用的所有配置。 Kubernetes Operator 监视 CR 类型并采取特定于应用的操作，确保当前状态与该资源的理想状态相符。通过自定义资源定义引入新的对象类型。Kubernetes API 可以像处理内置对象一样处理自定义资源定义，包括通过 kubectl 交互以及包含在基于角色的访问权限控制（RBAC）策略中。Operator 会持续监控正在运行的应用，可备份数据，从故障中恢复，以及随着时间的推移自动升级应用。 

Kubernetes Operator **几乎可执行任何操作**：扩展复杂的应用，应用版本升级，甚至使用专用硬件管理计算集群中节点的内核模块。

工作原理如下图所示：

![img](https://static001.infoq.cn/resource/image/85/2c/854b0a8eb938905e2e01195e55f5dd2c.png)

注：via [infoq Kubernetes Operator 基础入门](https://www.infoq.cn/article/3jrwfyszlu6jatbdrtov)

# 控制器模型

控制器就是保证系统按期望状态运行。下图展示的是整体架构，自定义控制器包含Informer、workQueue、Control Loop。

![img](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/k8s/k8s_custom_controller.png)

声明式API对象与控制器模型相辅相成，声明式API对象定义出期望的资源状态，实际状态往往来自于 Kubernetes 集群本身，比如**kubelet 通过心跳汇报的容器状态和节点状态**，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息。控制器模型则通过控制循环（Control Loop）将Kubernetes内部的资源调整为声明式API对象期望的样子。因此可以认为声明式API对象和控制器模型，才是Kubernetes项目编排能力“赖以生存”的核心所在。

![img](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/k8s/k8s_controller_definition.PNG)

详细控制器介绍可以参考：[Kubernetes 控制器模型](https://qiankunli.github.io/2019/03/07/kubernetes_controller.html)

## Reconcile 调协

调协 - reconcile，用来调控控制器的操作。例如，添加自定义控制器，核心处理逻辑就是Reconcile函数中。

在Kubernetes中，Pod是调度的基本单元，也是所有内置Workload管理的基本单元，无论是Deployment还是StatefulSet，它们在对管理的应用进行更新时，都是以Pod为单位。**所谓编排，最终落地就是 更新pod 的spec ,condition,container status 等数据**（原地更新或重建符合这些配置的pod）。非基本单位的 Deployment/StatefulSet 的变更更多是数据的持久化。

# controller-runtime

> 项目地址：[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)

该包提供用于构建控制器的库。控制器实现Kubernetes API，是构建Operator、工作负载api、配置api、自动伸缩器等的基础。

## 功能介绍

#### Controller

Controller实现了Kubernetes API来响应事件（对象创建/更新/删除），并确保对象Spec中指定的状态与系统状态匹配，这称之为**调协（reconcile）**。如果状态不匹配，Controller将根据需要创建/更新/删除对象以使它们匹配。

Controller通过实现工作队列来处理 reconcile.Requests （调协特定对象状态的请求）。
与HTTP处理流程不同，控制器**不**直接处理事件，而是将请求入队调协对象最终状态。这意味着针对多个事件可能是批处理的，并且每次做调协都需要了解当前系统状态。

* Controller需要一个Reconciler来从工作队列中取事件。
* Controller需要配置一个监听入队reconcile.Requests事件的响应。

#### Reconciler 调协

Reconciler是一个提供给Controller的函数，可以随时使用对象的Name和Namespace调用它。当被调用时，Reconciler将确保系统的状态与调用Reconciler时对象中指定的状态相匹配。
示例：为 ReplicaSet 对象调用的Reconciler。ReplicaSet指定了5个副本，但是系统中只有3个Pods。Reconciler创建了另外两个Pods，并将它们的OwnerReference设置为指向ReplicaSet，并使controller=true。

* Reconciler包含了Controller的所有业务逻辑。
* Reconciler通常工作于单一对象类型。e.g，它只协调ReplicaSets。对于不同的类型，使用不同的Controller。如果您希望从其他对象触发调协，可以提供一个映射(e.g，所有者的引用)，将触发调协的对象映射到被调协的对象。
* Reconciler提供了对象的Name / Namespace来进行调协。
* Reconciler不关心触发调协的事件内容或事件类型。e.g，无论ReplicaSet是创建还是更新，Reconciler总是会将系统中的Pods数量与对象在被调用时指定的数量进行比较。


# 引用


[1] 什么是 Kubernetes Operator，[https://www.redhat.com/zh/topics/containers/what-is-a-kubernetes-operator](https://www.redhat.com/zh/topics/containers/what-is-a-kubernetes-operator)

[2] Kubernetes Operator 基础入门，[https://time.geekbang.org/column/article/40583](https://time.geekbang.org/column/article/40583)

[3] 编排其实很简单：谈谈“控制器”模型，[https://time.geekbang.org/column/article/40583](https://time.geekbang.org/column/article/40583)

[4] Kubernetes 控制器模型 ，[https://qiankunli.github.io/2019/03/07/kubernetes_controller.html](https://qiankunli.github.io/2019/03/07/kubernetes_controller.html)
