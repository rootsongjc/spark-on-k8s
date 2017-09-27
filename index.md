---
layout: global
displayTitle: Apache Spark on Kubernetes
title: Apache Spark on Kubernetes
description: Apache Spark on Kubernetes 用户文档
---

### 概览

本网页是使用 Kubernetes 原生调度的 Apache Spark 用户文档。

GitHub repo：[apache-spark-on-k8s/spark](https://github.com/apache-spark-on-k8s/spark) ，直接从 Apache Spark 的官方 GitHub repo 中 fork 而来。

### 缘起

见 [SPARK-18278](https://issues.apache.org/jira/browse/SPARK-18278)。

这个 Issue 的目标是将 kubernetes 作为 spark 的原生调度器，跟 spark 的其他调度后台如 Standalone、Mesos、Yarn 是平级的。

### 为何使用 Spark on Kuberentes

使用 kubernetes 原生调度的 spark on kubernetes 是对原有的 spark on yarn 革命性的改变，主要表现在以下几点：

1. Kubernetes 原生调度：不再需要二层调度，直接使用 kubernetes 的资源调度功能，跟其他应用共用整个 kubernetes 管理的资源池；
2. 资源隔离，粒度更细：原先 yarn 中的 queue 在 spark on kubernetes 中已不存在，取而代之的是 kubernetes 中原生的 namespace，可以为每个用户分别指定一个 namespace，限制用户的资源 quota；
3. 细粒度的资源分配：可以给每个 spark 任务指定资源限制，实际指定多少资源就使用多少资源，因为没有了像 yarn 那样的二层调度（圈地式的），所以可以更高效和细粒度的使用资源；
4. 监控的变革：因为做到了细粒度的资源分配，所以可以对用户提交的每一个任务做到资源使用的监控，从而判断用户的资源使用情况，所有的 metric 都记录在数据库中，甚至可以为每个用户的每次任务提交计量；
5. 日志的变革：用户不再通过 yarn 的 web 页面来查看任务状态，而是通过 pod 的 log 来查看，可将所有的 kuberentes 中的应用的日志等同看待收集起来，然后可以根据标签查看对应应用的日志；

所有这些变革都能帮助我们更高效的获的、有效的利用资源，提高生产效率。


### 内容

* [在 Kubernetes 上运行 Spark](./running-on-kubernetes.html)
* [在云环境上运行 Spark](./running-on-kubernetes-cloud.html)
* [Spark 属性配置](./spark-properties.html)
* [用户指南](user-guide.html)
* [贡献说明](./contribute.html)
