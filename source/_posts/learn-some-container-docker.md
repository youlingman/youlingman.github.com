---
title: 容器学习系列-初识docker
tags:
  - container
  - docker
date: 2020-03-01 19:19:39
---
对容器和devops感兴趣已久，但是之前犯懒加上工作上没机会接触，一直没认真去了解，最近乘着长假，就着[《深入浅出Docker》](https://book.douban.com/subject/30486354/)和[菜鸟教程](https://www.runoob.com/docker/docker-tutorial.html)开始入门Docker，也在自己破电脑上耍了下单机版，记录一下学习小结。
<!--more-->

### 回顾：ops与IaaS

以前在实验室管刀片服务器的时候，需要先在服务器裸机上装vmware，然后通过vmware来加载镜像（操作系统镜像），再新增和管理实例，本质上是通过hypervisor层来管理物理服务器资源，向上提供节点实例（类比实际的物理机器节点），打比方把服务器的计算资源比作一个大蛋糕，hypervisor层负责切分出并管理一个个小蛋糕。

这种模式最典型的例子是VPS零售商，向用户交付虚拟主机裸机。从ops的角度，不方便的地方一个是基于单一物理服务器，扩展和迁移不太灵活，一个是管理上没有什么现成运维管理工具需要额外定制开发，还有就是虚拟机镜像的更新和维护也十分笨重。而从dev角度来看，用户需要额外去搞定开发/运行相关配置，如装依赖、下代码、跑通环境等等。

后来调研鼓捣了下openstack（folsom/grizly版本），openstack是在hypervisor之上的IaaS管理平台，与普通hypervisor的主要不同在于：

1. 提供一站式的IaaS服务；
2. 显式划分出计算、网络、存储几种核心资源的管理组件，hypervisor归入计算组件下管理；
3. 面向现有服务器集群部署，根据节点部署的对应openstack服务，划分为管理节点、计算节点、网络节点、存储节点，同一个节点可以身兼数职；

这种模式典型的是AWS、阿里云等云服务商，提供比较全面的基础设施资源管理能力，但是笨重的镜像问题仍然存在，而且如果是基于现有集群再启新VM的方式，性能花销不小。开发人员拿到VM实例后也还是要去重复的调开发/运行环境。

### docker初览

随着容器技术的发展，docker横空出世。docker带来一种轻量级VM-容器的概念，容器介乎虚拟机和进程之间，相对于虚拟机来说容器的启动和部署更快速灵活，而相对于进程来说容器还提供了对进程运行环境的封装。docker的优点主要来自以下两点特性：

**内核共享的隔离技术**

docker容器和普通虚拟机最大的不同，在于容器实例是利用共享内核的隔离技术创建出来的，而普通VM实例则是基于（硬件）虚拟化技术创建出来的。（硬件）虚拟化创建出来的VM实例本质上是在物理设备上运行的一个个独立的OS，而通过docker创建出来的容器实例，则可以理解为通过进程空间隔离、资源限制、目录结构隔离创建出沙盒环境，然后在该沙盒内挂载对应镜像的文件系统，从而提供一个运行实例。通过共享宿主OS内核，容器的运行开销相对于虚拟化VM来说有所降低，因此容器实例可以提供进程级别的启动速度和性能。

**多层结构镜像与标准化构建**

容器镜像本质上是对运行环境文件系统的打包，docker的镜像为分层结构，类比面向对象的父类继承结构，一个docker镜像可能由多层组成，不同镜像间可能会共享某些特定镜像层，基于分层结构，镜像的更新可以实现增量更新，每次更新只需要下载差异的部分，这可以大大加快镜像分发和容器部署迁移。

同时通过Dockerfile构建形式，docker可以提供统一的镜像封装机制，以往ops积累的shell脚本、perl脚本、python脚本，都可以统一纳入Dockerfile来管理维护，Dockerfile本身作为部署计划的工程化成果也可以纳入版本管理，对应构建出来的容器镜像则作为部署动作的前提。这个和[Google SRE](https://book.douban.com/subject/26875239/)的运维工程化以及DevOps理念是一致的。

利用容器镜像来交付应用，docker可以为迁移和部署（尤其是跨环境情况）的一致性提供保证，加上容器本身的隔离性，这个可以大大减轻开发和部署人员的心理压力。

### 容器的应用

从开发场景来看，如果是个人/小团队的原型开发情况，感觉docker很适合作为部署工具纳入持续集成流程来快速部署验证应用，同时也可以为实际部署积累工程化成果。

而如果是大型项目开发，则要考虑系统本身（或部分组件）是否适合容器化。对于应用开发者而言，容器实例更接近于一个运行中的进程和其对应运行环境（文件系统）的快照，将容器实例类比为一个运行在独立节点（与其它组件进程没有必须处于同一节点的约束条件）上且对外暴露端口的进程可能对于开发工程师来说更好理解。在决定将应用系统容器化前，要先问自己几个问题：

1. 系统考虑作容器化的相关组件，是否正在以上述形式运行；
2. 如果不是，是否可以改造成以上述形式运行；
3. 容器化改造后，系统是否能保持现有部署运行模式的监控、容错、灾备保障水平；
4. 系统是否长期存在高频率的部署需求；
5. 系统是否长期存在跨环境的部署/迁移需求；

感觉对于可以被拆分出来的无状态服务，容器化是值得考虑的一个方案。关于容器集群部署和编排部分，以后有时间再学习下k8s。