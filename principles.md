# 标准容器的5个原则

定义软件交付的单位为标准容器。标准容器的目标是将软件组件及其所有依赖（all its dependencies）以自描述和可移植的格式封装，这样任何兼容的运行时（runtime）都可以无需额外的依赖（without extra dependencies）地运行它，而不管底层机器和容器的内容。

标注容器规范定义：

1. 配置文件格式
2. 标准操作集
3. 执行环境

（对标准容器来说）一个伟大的比喻是运输行业里的物理集装箱。集装箱是交付的基本单位，它可以被抬起来（lifted），堆放（stacked），locked，装卸（loaded/unloaded）和贴标签（labeled）。Irrespective of their contents不论其内容, by standardizing the container itself it allowed for a consistent, more streamlined and efficient set of processes to be defined. For software Standard Containers offer similar functionality by being the fundamental, standardized, unit of delivery for a software package.

## 标准操作（Standard operations）

标准容器定义了一组“标准操作”。它们（容器）可以通过标准容器工具被创建，启动和停止；通过标准文件系统工具被复制（copied）和快照（snapshotted）；通过标准网络工具被下载和上传。

## 内容无关（Content-agnostic）

标准容器是内容无关的：无论内容如何，所有的标准操作都会得到相同的结果。它们以相同的方式启动，不论包含的是一个pg数据库，一个包含了依赖和应用服务器的php应用或是一个Java的构建。

## 基础设施无关（Infrastructure-agnostic）

标准容器是基础设施无关的：它们可以被运行在任何支持OCI的基础设施上。比如，一个标准容器可以被捆绑（bundled？）在笔记本电脑里，上传到云存储中，并由位于弗吉尼亚酒店中的构建服务器下载，运行或者快照，上传到自制私有云的10个staging服务器上，随后发送到三个共有云region的30个生产实例上。

## 为自动化设计（Designed for automation）

标准容器为自动化设计：因为他们提供了相同的标准操作，不管内容和基础设置，都非常适合自动化。事实上，你可以说自动化是它们的秘密武器。

许多曾经耗时，容易出错的人工都可以被程序化。在标准容器之前，一个软件组件到生产环境里运行之前，它被10个不同的人在10台不同的电脑上单独构建，配置，bundled，记录，修补，补充，模板化，调整和检验。构建失败，库冲突，镜像损坏，便利贴（post-it notes？）丢了，日志放错地方，集群升级失败。这个过程缓慢，低效，成本巨大 —— 根据（编程）语言和基础设施提供商的不同，这一过程完全不同。

## 工业级交付（Industrial-grade delivery）

标准容器使软件的工业级交付成为现实。利用上述所有的属性，标准容器使大，小型企业能够简化和自动化其软件交付途径。不论是内部的devops流程，还是外部的基于客户的软件交付机制，标准容器正在改变社区对软件打包和交付的想法。
