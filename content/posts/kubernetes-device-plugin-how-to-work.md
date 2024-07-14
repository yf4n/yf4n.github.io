+++
title = "Kubernetes DevicePlugin简介"
date = "2024-07-14"
description = "Kubernetes DevicePlugin简介"
tags = ["Kubernetes"]
categories = ["技术探讨"]
toc = true
+++

DevicePlugin是Kubernetes提供的扩展能力之一，那么Kubernetes中DevicePlugin是什么、有什么作用，又是如何进行工作的？
<!--more-->

## DevicePlugin简介
对于DevicePlugin，Kubernetes官网是这样介绍的
> Kubernetes 提供了一个设备插件框架，你可以用它来将系统硬件资源发布到 Kubelet。
> 供应商可以实现设备插件，由你手动部署或作为 DaemonSet 来部署，而不必定制 Kubernetes 本身的代码。目标设备包括 GPU、高性能 NIC、FPGA、 InfiniBand适配器以及其他类似的、可能需要特定于供应商的初始化和设置的计算资源。

Kubernetes提供了Device Plugins扩展机制，该机制可有效支撑CPU、内存之外的硬件资源扩展的需求。DevicePlugin包含`资源发现`和`资源分配`2个能力。`资源发现`主要是感知节点上的硬件资源情况并上报至集群中，允许Pod申请该资源进行使用。`资源分配`则是在Pod调度至节点后Kubelet通过调用DevicePlugin接口为Pod分配硬件资源的能力。

## DevicePlugin工作流程
DevicePlugin工作流程中，主要涉及Kubelet和DevicePlugin2个组件，二者均提供了一个gRPC Server，并按照约定暴露了一系列接口进行交互，其包含函数如图所示。
![kubelet-deviceplugin-1](./kubelet-and-deviceplugin-func.png)

### Kubelet
Kubelet中的DevicePluginManager模块的gRPC服务中仅包含一个`Register`接口，该接口主要用于DevicePlugin注册自身情况使用，所接受参数包含`Version`、`Endpoint`、`ResourceName`和`Options`。

### DevicePlugin
DevicePlugin的gRPC服务提供了共5个函数，分别是`ListAndWatch`、`PreStartContainer`、`Allocate`、`GetPreferredAllocation`、`GetDevicePluginOptions`，其各自作用如下所示

- ListAndWatch: 该接口会立即返回当前设备列表信息，之后保持连接，当设备发生变化时推送最新的设备列表信息。
- GetDevicePluginOptions: 返回该DevicePlugin是否支持`GetPreferredAllocation`和`PreStartContainr`两个接口。
- PreStartContainer: 在容器启动之前被调用，一般用于进行一些重置操作，如分配存储设备后对分配的硬件设备进行格式化重置之类操作。
- GetPreferredAllocation: 该接口主要用于在分配阶段询问DevicePlugin的偏好设备，其请求参数包含`AvailableDeviceIDs`、`MustIncludeDeviceIDs`、`AllocationSize`，分别代表可用设备ID列表、必须包含的设备ID列表(如容器重启前已用的设备)、所需设备数量。
- Allocate: 在容器创建期间调用，其请求参数包含要申请的设备ID，完成申请后返回申请到的设备信息，包含启动容器用的环境变量、卷挂载信息、注解信息等内容。
 
### 工作流程
由于不同的DevicePlugin设计上存在差异，这里以NVIDIA/k8s-device-plugin为例来描述该过程，要注意不同DevicePlugin设计可能不一样，流程上也可能存在不同。

![kubelet-deviceplugin-1](./kubelet-and-deviceplugin-workflow.png)

DevicePlugin与Kubelet之间的交互流程如下
1. Kubelet启动，清除/var/lib/kubelet/device-plugins/目录下所有文件，Kubelet中的DevicePluginManager启动gRPC Server，通过unix sock进行通信，监听/var/lib/kubelet/device-plugins/kubelet.sock。该流程会发生在Kubelet第一次启动或重启，清除目录下文件后所有DevicePlugin与Kubelet的连接均会中断，一般来说DevicePlugin均需要重新注册自身。
2. DevicePlugin感知到kubelet.sock文件存在后，通过kubelet.sock与DevicePluginManager建立连接，请求DevicePluginManager的`Register`接口注册自身。
3. DevicePluginManager接收到Register请求后，会根据Register中的信息建立与DevicePlugin的链接，并请求ListAndWatch接口获取监听节点设备列表信息。同时也会请求DevicePlugin的`GetDevicePluginOptions`接口获取对`GetPreferredAllocation`和`PreStartContainr`接口的支持情况。
4. 当有Pod调度到该节点上需要申请该节点的硬件资源时（此时为GPU），Kubelet通过建立的链接依次请求DevicePlugin的`GetPreferredAllocation`、`Allocate`、`PreStartContainr`接口，完成获取偏好硬件设备、申请硬件设备和初始化硬件设备三个工作。