---
layout: global
title: 在 Kubernetes 上运行 Spark
---

支持在 [Kubernetes](https://kubernetes.io/docs/whatisk8s/) 上运行 spark 目前还处于实验状态。目前的特性还没有在 kuberentes 集群上做很好的测试，运行起来还有很多的限制，请大家在谨慎考虑在生产环境下使用。

## 先决条件

- 您必须要有一个 kubernetes 集群，并且能通过 [kubectl](https://kubernetes.io/docs/user-guide/prereqs/) 命令访问到。如果在本地测试，你需要在自己的本地环境下安装 [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)。
  - 我们建议安装最新的 minikube 版本（本文档中使用的0.19.0版本），老版本中可能缺少一些必要组件。
- 您必须要有在集群中执行 create 和 list [pod](https://kubernetes.io/docs/user-guide/pods/)、[ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/) 还有 [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 的权限。您可以使用以下命令来验证： `kubectl get pods`、 `kubectl get configmaps`、 `kubectl get secrets` ，确定是否可以获取这些对的信息。
- 您必须有一个支持 Kuberentes 的 spark 发行版。可以通过以下两种方式获取：
  - [release tarball](https://github.com/apache-spark-on-k8s/spark/releases) 下载编译好的软件包
  - [自行编译支持 kuberentes 的 spark 发行版](https://github.com/apache-spark-on-k8s/spark/blob/branch-2.2-kubernetes/resource-managers/kubernetes/README.md#building-spark-with-kubernetes-support)

**我们接下来都会基于 kuberentes 1.6 版本来测试。**

## Docker 镜像

因为向 kubernetes 中提交的任务必须要有镜像，spark 程序要运行在 pod 中，也必须提供镜像，我们使用 docker 来构建镜像和作为容器的运行时环境。

如果您想要用已经提前编译好的 docker 镜像，可以使用 docker hub 中的镜像：[kubespark](https://hub.docker.com/u/kubespark/) 如下：

<table class="table">
<tr><th>组件</th><th>镜像</th></tr>
<tr>
  <td>Spark Driver Image</td>
  <td><code>kubespark/spark-driver:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
<tr>
  <td>Spark Executor Image</td>
  <td><code>kubespark/spark-executor:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
<tr>
  <td>Spark Initialization Image</td>
  <td><code>kubespark/spark-init:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
<tr>
  <td>Spark Staging Server Image</td>
  <td><code>kubespark/spark-resource-staging-server:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
<tr>
  <td>PySpark Driver Image</td>
  <td><code>kubespark/driver-py:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
<tr>
  <td>PySpark Executor Image</td>
  <td><code>kubespark/executor-py:v2.2.0-kubernetes-0.4.0</code></td>
</tr>
</table>

### 镜像版本说明

**0.4.0**

- Spark-submit 提交任务时不支持指定 kubernetes 的 serviceaccount，当 kubernetes 集群启用了 RBAC 的时候，将会出现权限错误。

### 构建镜像

您也可以根据需要定制化您的镜像，然后自行编译成 docker 镜像。在我们的 spark release 的 `dockerfiles` 目录中也包含了 Dockerfile 文件。

```bash
.
├── driver
│   └── Dockerfile
├── driver-py
│   └── Dockerfile
├── executor
│   └── Dockerfile
├── executor-py
│   └── Dockerfile
├── init-container
│   └── Dockerfile
├── resource-staging-server
│   └── Dockerfile
├── shuffle-service
│   └── Dockerfile
└── spark-base
    ├── Dockerfile
    └── entrypoint.sh
```

您可以手动编译这些镜像然后 push 到自己的镜像仓库中。

例如，加入您的镜像仓库地址是 `registry-host` 监听端口为 5000，可以使用下面的额命令编译和 push 镜像：

```bash
cd $SPARK_HOME
docker build -t registry-host:5000/spark-driver:latest -f dockerfiles/driver/Dockerfile .
docker build -t registry-host:5000/spark-executor:latest -f dockerfiles/executor/Dockerfile .
docker build -t registry-host:5000/spark-init:latest -f dockerfiles/init-container/Dockerfile .
docker push registry-host:5000/spark-driver:latest
docker push registry-host:5000/spark-executor:latest
docker push registry-host:5000/spark-init:latest
```

也可以使用该脚本 `https://github.com/apache-spark-on-k8s/spark/blob/branch-2.2-kubernetes/sbin/build-push-docker-images.sh` 来着自动编译和 push 镜像。

## 向kubernetes提交任务

使用 `spark-submit` 向 kubernetes 中提交 spark 任务。例如，使用上文列举的镜像，提交一个 spark pi 计算任务的命令如下：

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar
```

可以在 `spark-submit` 命令行中使用 `--master` 来指定 spark master，也可以在应用程序的配置中设置 `spark.master` 的地址，但是 URL 必须按照这种格式 `k8s://<api_server_url>`。使用 `k8s://` 前缀指定向 kubernetes 集群上提交 spark 任务， API server 的地址是 `api_server_url`。如果不指定 HTTP 协议的话默认使用的是 `https`。例如指定 master 为 `k8s://example.com:443` ，等效于 `k8s://https://example.com:443`，如果您是用的是未启用 TLS 的其他端口，master 地址应该这样指定： `k8s://http://example.com:8443`.

使用该的命令来确定 API server 的地址： `kubectl cluster-info`。

```bash
> kubectl cluster-info
Kubernetes master is running at http://127.0.0.1:8080
```

对于该集群可以使用使用该地址提及任务：`--master k8s://http://127.0.0.1:8080`

目前 spark driver 和 executor 都只能在 cluster mode 下运行，全部都作为 pod 运行在 kubernetes 集群中。

**注意：**在上面的提交命令中我们使用 `local://` 格式指定了一个 jar 文件，这个 **local** 的实际意义是该 jar 文件位于 docker 镜像的某个目录下，而不是您提交任务的那台主机的某个目录下的 jar 文件。如何提交本地的 jar 文件将在下文的 **依赖管理** 中讨论。

## Python 支持

随着 Python 在数据科学领域的广泛应用，我们增加了 PySpark 的支持。
These applications follow the general syntax that you would expect from other cluster managers. The submission of a PySpark job is similar to the submission of Java/Scala applications except you do not supply a class, as expected. 
Here is how you would execute a Spark-Pi example:

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace <k8s-namespace> \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/driver-py:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/executor-py:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  --jars local:///opt/spark/examples/jars/spark-examples_2.11-2.2.0-k8s-0.4.0-SNAPSHOT.jar \
  local:///opt/spark/examples/src/main/python/pi.py 10
```

With Python support it is expected to distribute `.egg`, `.zip` and `.py` libraries to executors via the `--py-files` option. 
​We support this as well, as seen with the following example:   

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace <k8s-namespace> \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/driver-py:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/executor-py:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  --jars local:///opt/spark/examples/jars/spark-examples_2.11-2.2.0-k8s-0.4.0-SNAPSHOT.jar \
  --py-files local:///opt/spark/examples/src/main/python/sort.py \
  local:///opt/spark/examples/src/main/python/pi.py 10
```

You may also customize your Docker images to use different `pip` packages that suit your use-case. As you can see with the current `driver-py` Docker image we have commented out the current pip module support that you can uncomment to use:

```dockerfile
...
ADD examples /opt/spark/examples
ADD python /opt/spark/python

RUN apk add --no-cache python && \
    python -m ensurepip && \
    rm -r /usr/lib/python*/ensurepip && \
    pip install --upgrade pip setuptools && \
    rm -r /root/.cache
# UNCOMMENT THE FOLLOWING TO START PIP INSTALLING PYTHON PACKAGES
# RUN apk add --update alpine-sdk python-dev
# RUN pip install numpy
...
```

And bake into your docker image whichever PySpark files you wish to include by merely appending to the following exec command with your appropriate file (i.e. MY_SPARK_FILE)

```dockerfile
...
CMD SPARK_CLASSPATH="${SPARK_HOME}/jars/*" && \
    if ! [ -z ${SPARK_MOUNTED_CLASSPATH+x} ]; then SPARK_CLASSPATH="$SPARK_MOUNTED_CLASSPATH:$SPARK_CLASSPATH"; fi && \
    if ! [ -z ${SPARK_SUBMIT_EXTRA_CLASSPATH+x} ]; then SPARK_CLASSPATH="$SPARK_SUBMIT_EXTRA_CLASSPATH:$SPARK_CLASSPATH"; fi && \
    if ! [ -z ${SPARK_EXTRA_CLASSPATH+x} ]; then SPARK_CLASSPATH="$SPARK_EXTRA_CLASSPATH:$SPARK_CLASSPATH"; fi && \
    if ! [ -z ${SPARK_MOUNTED_FILES_DIR} ]; then cp -R "$SPARK_MOUNTED_FILES_DIR/." .; fi && \
    exec /sbin/tini -- ${JAVA_HOME}/bin/java $SPARK_DRIVER_JAVA_OPTS -cp $SPARK_CLASSPATH \
    -Xms$SPARK_DRIVER_MEMORY -Xmx$SPARK_DRIVER_MEMORY \
    $SPARK_DRIVER_CLASS $PYSPARK_PRIMARY MY_PYSPARK_FILE,$PYSPARK_FILES $SPARK_DRIVER_ARGS
```

## 依赖管理

我们这里所说的依赖管理主要是指 Jar 包的依赖管理。

应用程序以来需要从你的本机提交到 **resource staging server**，这样 driver 和 executor 才能从中获取依赖的文件。运行 resource staging server 的 YAML 文件可以在这里找到： `conf/kubernetes-resource-staging-server.yaml`。

该 YAML 文件中定义个只运行一个 resource staging server pod 的 Deployment 对象，同时挂载了一个 ConfigMap，然后定义了一个 service 通过 NodePort 的方式暴露到集群外部。

使用默认的配置创建一个 resource staging server：

```
kubectl create -f conf/kubernetes-resource-staging-server.yaml
```

这样你就可以在 `spark-submit` 命令中指定使用 `--conf spark.kubernetes.resourceStagingServer.uri` 参数来指定 resource staging server 的地址了：

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.resourceStagingServer.uri=http://<address-of-any-cluster-node>:31000 \
  examples/jars/spark_examples_2.11-2.2.0.jar
```

Resource staging server 的镜像您也可以通过 Dockerfile 来从源码构建： `dockerfiles/resource-staging-server/Dockerfile`。

我们在上面的命令中使用了 NodePort 将 service 暴露到集群外部，请确保集群上宿主机的 Node 节点的 31000 端口没有被占用，如果该端口被占用就换一个其他端口，您可以通过任何一个 node 节点加上 31000 端口即可访问到 resource staging server。

### 不使用 Resource Staging Server 做依赖管理

仅当您需要提交本地依赖文件的时候才需要用到 resource staging server，如果您的应用程序依赖全部托管在远程比如 HDFS 或者 http server 的时候就不需要用到它。当然您可以把这些依赖编译到 docker 镜像里面，然后在执行 `spark-submit` 的时候通过 `local://` 指定依赖的文件或者在 Dockerfile 中设置 `SPARK_EXTRA_CLASSPATH` 环境变量。

### 访问 Kubernetes 集群

还可以通过 [local kubectl proxy](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#using-kubectl-proxy) 执行 spark-submit。可以使用 proxy 来直接跟 API server 交互而不用给 spark-submit 传递认证信息。

启动本地的 proxy：

```bash
kubectl proxy
```

如果您的本地 proxy 监听 8001 端口，我们像这样提交任务：

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://http://127.0.0.1:8001 \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar
```

Spark 跟 Kubernetes 集群之间交互使用的是 fabric8 的 kubernetes-client library。

当我们使用的是 fabric8  的 kubernetes-client 所不支持的认证机制时可以使用 `kubectl proxy`。它目前支持X509 Client Certs 和 OAuth tokens。

## 动态 Executor Scale

Spark on Kubernetes 支持 cluster mode 下的动态分配。该模式需要运行一个外部 shuffle service，通常是以一个注入 [hostpath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) 的 [daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 的方式来运行。该 shuffle service 可以在不同的 spark jobs 之间共享。在 kubernetes 中使用 spark 的动态分配功能，集群管理员必须在集群中启动一个或者多个 shuffle-service daemonset。

在 `conf/kubernetes-shuffle-service.yaml` 目录下有一个 shuffle service 的配置，用户可以根据自己的集群来定制。注意**恰当的设置** `spec.template.metadata.labels` ，因为在集群中可能有多个 shuffle service 实例。该 Label 用于 spark 应用定位到不同的 shuffle service。

例如，我们想使用一个位于 default namespace 下的 shuffle service，pod 的 label 有 `app=spark-shuffle-service` 和 `spark-version=2.2.0`，我们想要在提交任务时用这些 tag 来定位到某个 shuffle service。为了使用动态分配，我们提交任务时应该这样写：

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.GroupByTest \
  --master k8s://<k8s-master>:<port> \
  --kubernetes-namespace default \
  --conf spark.app.name=group-by-test \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:latest \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:latest \
  --conf spark.dynamicAllocation.enabled=true \
  --conf spark.shuffle.service.enabled=true \
  --conf spark.kubernetes.shuffle.namespace=default \
  --conf spark.kubernetes.shuffle.labels="app=spark-shuffle-service,spark-version=2.2.0" \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar 10 400000 2
```

注意其中 `spark.kubernetes.shuffle.labels` 的值。

## 高级

### 使用 TLS 加密 the Resource Staging Server

The default configuration of the resource staging server is not secured with TLS. It is highly recommended to configure this to protect the secrets and jars/files being submitted through the staging server.

The YAML file in `conf/kubernetes-resource-staging-server.yaml` includes a ConfigMap resource that holds the resource staging server's configuration. The properties can be adjusted here to make the resource staging server listen over TLS.
Refer to the [security](security.html) page for the available settings related to TLS. The namespace for the resource staging server is `kubernetes.resourceStagingServer`, so for example the path to the server's keyStore would be set by `spark.ssl.kubernetes.resourceStagingServer.keyStore`.

In addition to the settings specified by the previously linked security page, the resource staging server supports the following additional configurations:

<table class="table">
<tr><th>属性名称</th><th>默认值</th><th>含义</th></tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyPem</code></td>
  <td>(none)</td>
  <td>

Private key file encoded in PEM format that the resource staging server uses to secure connections over TLS. If this is specified, the associated public key file must be specified in
<code>spark.ssl.kubernetes.resourceStagingServer.serverCertPem</code>. PEM files and a keyStore file (set by
<code>spark.ssl.kubernetes.resourceStagingServer.keyStore</code>) cannot both be specified at the same time.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.serverCertPem</code></td>
  <td>(none)</td>
  <td>

Certificate file encoded in PEM format that the resource staging server uses to secure connections over TLS. If this is specified, the associated private key file must be specified in
<code>spark.ssl.kubernetes.resourceStagingServer.keyPem</code>. PEM files and a keyStore file (set by
<code>spark.ssl.kubernetes.resourceStagingServer.keyStore</code>) cannot both be specified at the same time.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyStorePasswordFile</code></td>
  <td>(none)</td>
  <td>

Provides the KeyStore password through a file in the container instead of a static value. This is useful if the
keyStore's password is to be mounted into the container with a secret.

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyPasswordFile</code></td>
  <td>(none)</td>
  <td>

Provides the keyStore's key password using a file in the container instead of a static value. This is useful if the keyStore's key password is to be mounted into the container with a secret.

  </td>
</tr>
</table>

Note that while the properties can be set in the ConfigMap, you will still need to consider the means of mounting the appropriate secret files into the resource staging server's container. A common mechanism that is used for this is to use [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) that are mounted as secret
volumes. Refer to the appropriate Kubernetes documentation for guidance and adjust the resource staging server's pecification in the provided YAML file accordingly.

Finally, when you submit your application, you must specify either a trustStore or a PEM-encoded certificate file to communicate with the resource staging server over TLS. The trustStore can be set with
`spark.ssl.kubernetes.resourceStagingServer.trustStore`, or a certificate file can be set with
`spark.ssl.kubernetes.resourceStagingServer.clientCertPem`. For example, our SparkPi example now looks like this:

```bash
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.4.0 \
  --conf spark.kubernetes.resourceStagingServer.uri=https://<address-of-any-cluster-node>:31000 \
  --conf spark.ssl.kubernetes.resourceStagingServer.enabled=true \
  --conf spark.ssl.kubernetes.resourceStagingServer.clientCertPem=/home/myuser/cert.pem \
  examples/jars/spark_examples_2.11-2.2.0.jar
```

### Spark Properties

下面是单独针对 spark on kubernetes 的一些配置。其他配置项就跟使用 YARN 或 Mesos 运行一样。查看 [配置页面](configuration.html) 获取更多信息。

<table class="table">
<tr><th>属性名称</th><th>默认值</th><th>含义</th></tr>
<tr>
  <td><code>spark.kubernetes.namespace</code></td>
  <td><code>default</code></td>
  <td>

The namespace that will be used for running the driver and executor pods. When using
<code>spark-submit</code> in cluster mode, this can also be passed to <code>spark-submit</code> via the <code>--kubernetes-namespace</code> command line argument.

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

 Add the annotation specified by <code>AnnotationName</code> to the driver pod. For example, <code>spark.kubernetes.driver.annotation.something=true</code>.

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
 Add the environment variable specified by <code>EnvironmentVariableName</code> to
 the Executor process. The user can specify multiple of these to set multiple environment variables.
   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.driverEnv.[EnvironmentVariableName]</code></td> 
   <td>(none)</td>
   <td>
 Add the environment variable specified by <code>EnvironmentVariableName</code> to
 the Driver process. The user can specify multiple of these to set multiple environment variables.
   </td>
 </tr>
</table>

## 目前的限制

在 Kubernetes 上运行 spark 还处于实验状态。当前的实现中还有一些限制，可能会在未来解决：

- 应用程序只能在 cluster mode 下运行。
- 只支持 Scala、Java 和 Python 应用。
