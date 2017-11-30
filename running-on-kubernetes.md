---
layout: global
title: 在 Kubernetes 上运行 Spark
---

支持在 [Kubernetes](https://kubernetes.io/docs/whatisk8s/) 上运行 spark 目前还处于实验状态。目前的特性还没有在 kuberentes 集群上做很好的测试，运行起来还有很多的限制，请大家在谨慎考虑在生产环境下使用。

## 先决条件

- 您必须要有一个 kubernetes 集群，支持的最低版本为1.6，并且要安装了 [kube-dns](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- ) 并且能通过 [kubectl](https://kubernetes.io/docs/user-guide/prereqs/) 命令访问到。如果在本地测试，你需要在自己的本地环境下安装 [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)。
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

随着 Python 在数据科学领域的广泛应用，我们增加了 PySpark 的支持。提交 PySpark 任务跟提交 Java/Scala 应用程序类似，不过不需要再指定 class。 
我们这样执行 Spark-Pi 示例：

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

为了支持 Python，可以使用 `--py-files` 选项为 executor 指定分布式的 `.egg`、`.zip` 和 `.py` 库。

例如下面的示例：

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

您也可以自定义 Docker 使用不同的 `pip` 包来满足自己的使用场景。从当前得 `driver-py` Docker 镜像的 Dockerfile 中您可以看到我们注释掉了一些 pip 模块，如果您要使用的话可以取消注释。

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

不管想要引入什么 PySpark 文件，只要在 Dockerfile 中的 exec 位置追加（即 MY_SPARK_FILE）。

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

## 访问 Driver UI

使用 `kubectl port-forward` 来访问 spark Driver 的 UI。

```bash
kubectl port-forward <driver-pod-name> 4040:4040
```

然后，通过 http://localhost:4040 访问。


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

### 使用 TLS 加密 Resource Staging Server

Resource staging server 默认不启用 TLS 加密。我们建议您启用该配置以加密提交到 Resource Staging Server 的 jar 包和文件。

该文件`conf/kubernetes-resource-staging-server.yaml` 中包含了 Resource Staging Server 配置的 ConfigMap。可以在这里调整属性以使 Resource Staging Server 通过TLS进行侦听。请参阅 [安全](security.html) 页面获取关于 TLS 配置的更多信息。Resource Staging Server 的 namespace 是 `kubernetes.resourceStagingServer`，该 Server 的 keyStore 应该配置成`spark.ssl.kubernetes.resourceStagingServer.keyStore`。

除了上面安全链接中的配置之外，Resource Staging Server 还支持下列配置。

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

证书文件以PEM格式编码，Resource Staging Server 使用它来进行TLS保护连接。如果指定，则必须在

<code>spark.ssl.kubernetes.resourceStagingServer.keyPem</code> 中指定关联的私钥文件。 PEM 文件和 keyStore 文件（通过 <code>spark.ssl.kubernetes.resourceStagingServer.keyStore</code>) 来设置的不能同时设定。

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyStorePasswordFile</code></td>
  <td>(none)</td>
  <td>

通过文件提供 KeyStore 密码而不是通过静态值。这当 keyStore 的密码是通过 secret 挂载到容器中的时候很有用。
</td>

</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyPasswordFile</code></td>
  <td>(none)</td>
  <td>

用容器中的文件提供keyStore的密钥密码，而不是静态值。如果keyStore的密钥密码使用 secret 挂载到容器中的话这很有用。

  </td>
</tr>
</table>

Note that while the properties can be set in the ConfigMap, you will still need to consider the means of mounting the appropriate secret files into the resource staging server's container. A common mechanism that is used for this is to use [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) that are mounted as secret volumes. Refer to the appropriate Kubernetes documentation for guidance and adjust the resource staging server's pecification in the provided YAML file accordingly.

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

## 目前的限制

在 Kubernetes 上运行 spark 还处于实验状态。当前的实现中还有一些限制，可能会在未来解决：

- 应用程序只能在 cluster mode 下运行。
- 只支持 Scala、Java 和 Python 应用。
