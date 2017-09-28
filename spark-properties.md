---
layout: global
title: Spark属性配置
---

Spark on kubernetes 在原生 spark 的基础上又增加了一些对 kubernetes 支持的配置文件，在下面的配置中凡是属性中包含 **kubernetes** 的属性都是新增内容，只有在支持 kubernetes 的 spark 版本中才生效。

下面是单独针对 spark on kubernetes 的一些配置。其他配置项就跟使用 YARN 或 Mesos 运行一样。查看 [配置页面](https://spark.apache.org/docs/latest/configuration.html) 获取更多信息。

<table class="table">
<tr><th>属性名称</th><th>默认值</th><th>含义</th></tr>
<tr>
  <td><code>spark.kubernetes.namespace</code></td>
  <td><code>default</code></td>
  <td>

指定运行 driver 和 executor pod 的 namespace。当在 cluster mode 下使用<code>spark-submit</code>提交任务时，可以在命令行中增加 <code>--kubernetes-namespace</code> 参数。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.docker.image</code></td>
  <td><code>spark-driver:2.2.0</code></td>
  <td>

Driver docker 镜像。 使用标注的<a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> 格式。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.docker.image</code></td>
  <td><code>spark-executor:2.2.0</code></td>
  <td>

Executor docker 镜像。 使用标注的<a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> 格式。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.initcontainer.docker.image</code></td>
  <td><code>spark-init:2.2.0</code></td>
  <td>

Docker image to use for the init-container that is run before the driver and executor containers. Specify this using the standard <a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> format. The init-container is responsible for fetching application dependencies from both remote locations like HDFS or S3, and from the resource staging server, if applicable.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.shuffle.namespace</code></td>
  <td><code>default</code></td>
  <td>

Namespace in which the shuffle service pods are present. The shuffle service must be created in the cluster prior to attempts to use it.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.shuffle.labels</code></td>
  <td>(none)</td>
  <td>

Labels that will be used to look up shuffle service pods. This should be a comma-separated list of label key-value pairs, where each label is in the format <code>key=value</code>. The labels chosen must be such that they match exactly one shuffle service pod on each node that executors are launched.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.allocation.batch.size</code></td>
  <td><code>5</code></td>
  <td>

每一轮 executor pod 分配时启动的 pod 个数。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.allocation.batch.delay</code></td>
  <td><code>1</code></td>
  <td>

每一轮 executor pod 分配时等待的秒数。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.caCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the CA cert file for connecting to the Kubernetes API server over TLS when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

Path to the client key file for authenticating against the Kubernetes API server when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.clientCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the client cert file for authenticating against the Kubernetes API server when starting the driver. This file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.oauthToken</code></td>
  <td>(none)</td>
  <td>

OAuth token to use when authenticating against the Kubernetes API server when starting the driver. Note
that unlike the other authentication options, this is expected to be the exact string value of the token to use for the authentication.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.caCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the CA cert file for connecting to the Kubernetes API server over TLS from the driver pod when requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

Path to the client key file for authenticating against the Kubernetes API server from the driver pod when requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme). If this is specified, it is highly recommended to set up TLS for the driver submission server, as this value is sensitive information that would be passed to the driver pod in plaintext otherwise.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.clientCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the client cert file for authenticating against the Kubernetes API server from the driver pod when
requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.oauthToken</code></td>
  <td>(none)</td>
  <td>

OAuth token to use when authenticating against the against the Kubernetes API server from the driver pod when requesting executors. Note that unlike the other authentication options, this must be the exact string value of the token to use for the authentication. This token value is uploaded to the driver pod. If this is specified, it is highly recommended to set up TLS for the driver submission server, as this value is sensitive information that would be passed to the driver pod in plaintext otherwise.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.serviceAccountName</code></td>
  <td><code>default</code></td>
  <td>

Service account that is used when running the driver pod. The driver pod uses this service account when requesting executor pods from the API server. Note that this cannot be specified alongside a CA cert file, client key file, client cert file, and/or OAuth token.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.caCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the CA cert file for connecting to the Kubernetes API server over TLS from the resource staging server when it monitors objects in determining when to clean up resource bundles.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

Path to the client key file for authenticating against the Kubernetes API server from the resource staging server when it monitors objects in determining when to clean up resource bundles. The resource staging server must have credentials that allow it to view API objects in any namespace.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.clientCertFile</code></td>
  <td>(none)</td>
  <td>

Path to the client cert file for authenticating against the Kubernetes API server from the resource staging server when it monitors objects in determining when to clean up resource bundles. The resource staging server must have credentials that allow it to view API objects in any namespace.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.oauthToken</code></td>
  <td>(none)</td>
  <td>

OAuth token value for authenticating against the Kubernetes API server from the resource staging server
when it monitors objects in determining when to clean up resource bundles. The resource staging server must have credentials that allow it to view API objects in any namespace. Note that this cannot be set at the same time as
<code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code>.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code></td>
  <td>(none)</td>
  <td>

File containing the OAuth token to use when authenticating against the against the Kubernetes API server from the resource staging server, when it monitors objects in determining when to clean up resource bundles. The resource staging server must have credentials that allow it to view API objects in any namespace. Note that this cannot be set at the same time as <code>spark.kubernetes.authenticate.resourceStagingServer.oauthToken</code>.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.useServiceAccountCredentials</code></td>
  <td>true</td>
  <td>

Whether or not to use a service account token and a service account CA certificate when the resource staging server authenticates to Kubernetes. If this is set, interactions with Kubernetes will authenticate using a token located at
<code>/var/run/secrets/kubernetes.io/serviceaccount/token</code> and the CA certificate located at
<code>/var/run/secrets/kubernetes.io/serviceaccount/ca.crt</code>. Note that if
<code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code> is set, it takes precedence over the usage of the service account token file. Also, if
<code>spark.kubernetes.authenticate.resourceStagingServer.caCertFile</code> is set, it takes precedence over using the service account's CA certificate file. This generally should be set to true (the default value) when the resource staging server is deployed as a Kubernetes pod, but should be set to false if the resource staging server is deployed by other means (i.e. when running the staging server process outside of Kubernetes). The resource staging server must have credentials that allow it to view API objects in any namespace.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.memoryOverhead</code></td>
  <td>默认是 executor 内存 * 0.10，最小值是 384M</td>
  <td>

分配给每个 executor 的超过 heap 内存的值，作为附加开销，单位可以为k、m、g等。该值用于虚拟机的开销、其他本地服务开销。根据 executor 的大小设置（通常是 6%到10%）。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.label.[LabelName]</code></td>
  <td>(none)</td>
  <td>

Custom labels that will be added to the driver pod. This should be a comma-separated list of label key-value pairs, where each label is in the format <code>key=value</code>. Note that Spark also adds its own labels to the driver pod for bookkeeping purposes.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.annotation.[AnnotationName]</code></td>
  <td>(none)</td>
  <td>

使用 <code>AnnotationName</code> 为 driver pod 指定 annotation。例如 <code>spark.kubernetes.driver.annotation.something=true</code>。

</td>

</tr>
<tr>
  <td><code>spark.kubernetes.executor.label.[LabelName]</code></td>
  <td>(none)</td>
  <td>

Add the label specified by <code>LabelName</code> to the executor pods. For example, <code>spark.kubernetes.executor.label.something=true</code>. Note that Spark also adds its own labels to the driver pod for bookkeeping purposes.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.annotation.[AnnotationName]</code></td>
  <td>(none)</td>
  <td>

Add the annotation specified by <code>AnnotationName</code> to the executor pods. For example, <code>spark.kubernetes.executor.annotation.something=true</code>.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.pod.name</code></td>
  <td>(none)</td>
  <td>

Driver pod 的名字。如果未设置，driver pod 的名字将被设置为”spark.app.name“ 加上当前时间戳作为后缀，以避免冲突。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.submission.waitAppCompletion</code></td>
  <td><code>true</code></td>
  <td>

In cluster mode, whether to wait for the application to finish before exiting the launcher process.  When changed to false, the launcher has a "fire-and-forget" behavior when launching the Spark job.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.port</code></td>
  <td><code>10000</code></td>
  <td>

Resource staging server 部署后监听的端口。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.uri</code></td>
  <td>(none)</td>
  <td>

URI of the resource staging server that Spark should use to distribute the application's local dependencies. Note that by default, this URI must be reachable by both the submitting machine and the pods running in the cluster. If one URI is not simultaneously reachable both by the submitter and the driver/executor pods, configure the pods to access the staging server at a different URI by setting
<code>spark.kubernetes.resourceStagingServer.internal.uri</code> as discussed below.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.internal.uri</code></td>
  <td>Value of <code>spark.kubernetes.resourceStagingServer.uri</code></td>
  <td>

URI of the resource staging server to communicate with when init-containers bootstrap the driver and executor pods with submitted local dependencies. Note that this URI must by the pods running in the cluster. This is useful to set if the resource staging server has a separate "internal" URI that must be accessed by components running in the cluster.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStore</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.trustStore</code></td>
  <td>

Location of the trustStore file to use when communicating with the resource staging server over TLS, as
init-containers bootstrap the driver and executor pods with submitted local dependencies. This can be a URI with a scheme of <code>local://</code>, which denotes that the file is pre-mounted on the pod's disk. A uri without a scheme or a scheme of <code>file://</code> will result in this file being mounted from the submitting machine's disk as a secret into the init-containers.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStorePassword</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.trustStorePassword</code></td>
  <td>

Password of the trustStore file that is used when communicating with the resource staging server over TLS, as init-containers bootstrap the driver and executor pods with submitted local dependencies.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStoreType</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.trustStoreType</code></td>
  <td>

Type of the trustStore file that is used when communicating with the resource staging server over TLS, when init-containers bootstrap the driver and executor pods with submitted local dependencies.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.clientCertPem</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.clientCertPem</code></td>
  <td>

Location of the certificate file to use when communicating with the resource staging server over TLS, as
init-containers bootstrap the driver and executor pods with submitted local dependencies. This can be a URI with a scheme of <code>local://</code>, which denotes that the file is pre-mounted on the pod's disk. A uri without a scheme or a scheme of <code>file://</code> will result in this file being mounted from the submitting machine's disk as a secret into the init-containers.

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.mountdependencies.jarsDownloadDir</code></td>
  <td><code>/var/spark-data/spark-jars</code></td>
  <td>

下载 Jar 包到 driver 和 executor 中的路径。该路径将作为 empty dir volume 挂载到 driver 和 executor 容器中。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.mountdependencies.filesDownloadDir</code></td>
  <td><code>/var/spark-data/spark-files</code></td>
  <td>

下载文件到 driver 和 executor 中的路径。该路径将作为 empty dir volume 挂载到 driver 和 executor 容器中。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.report.interval</code></td>
  <td><code>1s</code></td>
  <td>

在 cluster  mode 下报告当前 spark job 状态的时间间隔。

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.docker.image.pullPolicy</code></td>
  <td><code>IfNotPresent</code></td>
  <td>

Kubernetes 中的 docker 镜像拉取策略。

  </td>
</tr>
<tr>
   <td><code>spark.kubernetes.driver.limit.cores</code></td>
   <td>(none)</td>
   <td>

指定 driver pod 的 hard cpu limit。

   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.executor.limit.cores</code></td>
   <td>(none)</td>
   <td>

指定单个 executor pod 的 hard cpu limit。

   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.node.selector.[labelKey]</code></td> 
   <td>(none)</td>
   <td>

 Adds to the node selector of the driver pod and executor pods, with key <code>labelKey</code> and the value as the  configuration's value. For example, setting <code>spark.kubernetes.node.selector.identifier</code> to <code>myIdentifier</code> will result in the driver pod and executors having a node selector with key <code>identifier</code> and value   <code>myIdentifier</code>. Multiple node selector keys can be added by setting multiple configurations with this prefix.
</td>

  </tr>
 <tr>
   <td><code>spark.executorEnv.[EnvironmentVariableName]</code></td> 
   <td>(none)</td>
   <td>
 通过 <code>EnvironmentVariableName</code> 为 Executor 进程指定环境变量。用户可以指定多个环境变量。
   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.driverEnv.[EnvironmentVariableName]</code></td> 
   <td>(none)</td>
   <td>
通过 <code>EnvironmentVariableName</code> 为 Driver 进程指定环境变量。用户可以指定多个环境变量。
   </td>
 </tr>
</table>