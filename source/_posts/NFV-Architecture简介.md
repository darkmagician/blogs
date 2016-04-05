---
title: NFV Architecture简介
date: 2016-03-23 21:09:59
tags: Cloud
---
NFV (Network Function Virtualization), 即网络功能虚拟化. 是电信行业网络虚拟化的一个标准。一般运营商往往在内部维护着庞大的网络系统，其中包括了很多网络服务器。利用云计算技术把这些服务运行于虚拟化环境中，可以实现物理资源的最大化利用率，灵活的部署策略。例如 PCRF，PGW， SGW等网络设备都可以运行于虚拟化环境中。

在NFV的Architecture Framework中包括：

1. Virtualised Network Function (VNF) 被虚拟化的网络服务。
1. Element Management (EM) 负责管理和监控VNF。
1. NFV Infrastructure 提供NFV虚拟化资源，VNF运行在之上。
1. Virtualised Infrastructure Manager 管理NFV Infrastructure。
1. NFV Orchestrator 提供NFV服务编排。
1. VNF Manager 控制VNF的生命周期, 比如起停VNF。
1. Service, VNF and Infrastructure Description VNF的描述文件，其中包括VNF对资源的要求，比如cpu，内存，网络，存储等， 也包括了如何新建和控制VNF instance, 以及对其他service的依赖。
1. Operations and Business Support Systems (OSS/BSS) 管理模块，其实就是包含了Orchestrator， Virtualised Infrastructure Manager， VNF Manager还有就是那些description

![Architecture](architecture.png)

各个模块直接定义了一些接口，包括几个management模块之间的交互，以及management模块与其相应功能模块的控制接口。


