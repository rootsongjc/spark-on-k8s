---
layout: global
title: 在 Kubernetes 上运行 Spark
---

支持在 [Kubernetes](https://kubernetes.io/docs/whatisk8s/) 上运行 spark 目前还处于实验状态。目前的特性还没有在 kuberentes 集群上做很好的测试，运行起来还有很多的限制，请大家在谨慎考虑在生产环境下使用。

## 先决条件

- 您必须要有一个 kubernetes 集群，并且能通过 [kubectl](https://kubernetes.io/docs/user-guide/prereqs/) 命令访问到。如果在本地测试，你需要在自己的本地环境下安装 [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)。
  - 我们建议安装最新的 minikube 版本（本文档中使用的0.19.0版本），老版本中可能缺少一些必要组件。
- 您必须要有在集群中执行 create 和 list [pod](https://kubernetes.io/docs/user-guide/pods/)、[ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/) 还有 [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 的权限。您可以使用以下命令来验证： `kubectl get pods`、 `kubectl get configmap`、 `kubectl get secrets` ，确定是否可以获取这些对的信息。
- 您必须有一个支持 Kuberentes 的 spark 发行版。可以通过以下两种方式获取：
  - [release tarball](https://github.com/apache-spark-on-k8s/spark/releases) 下载编译好的软件包
  - [自行编译支持 kuberentes 的 spark 发行版](https://github.com/apache-spark-on-k8s/spark/blob/branch-2.2-kubernetes/resource-managers/kubernetes/README.md#building-spark-with-kubernetes-support)

**我们接下来都会基于 kuberentes 1.6 版本来测试。**

## Driver & Executor 镜像

因为向 kubernetes 中提交的任务必须要有镜像，spark 程序要运行在 pod 中，也必须提供镜像，我们使用 docker 来构建镜像和作为容器的运行时环境。

如果您想要用已经提前编译好的 docker 镜像，可以使用 docker hub 中的镜像：[kubespark](https://hub.docker.com/u/kubespark/) 如下：

<table class="table">
<tr><th>Component</th><th>Image</th></tr>
<tr>
  <td>Spark Driver Image</td>
  <td><code>kubespark/spark-driver:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
<tr>
  <td>Spark Executor Image</td>
  <td><code>kubespark/spark-executor:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
<tr>
  <td>Spark Initialization Image</td>
  <td><code>kubespark/spark-init:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
<tr>
  <td>Spark Staging Server Image</td>
  <td><code>kubespark/spark-resource-staging-server:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
<tr>
  <td>PySpark Driver Image</td>
  <td><code>kubespark/driver-py:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
<tr>
  <td>PySpark Executor Image</td>
  <td><code>kubespark/executor-py:v2.1.0-kubernetes-0.3.1</code></td>
</tr>
</table>

### 镜像版本说明

**0.3.1**

- Spark-submit 提交任务时不支持指定 kubernetes 的 serviceaccount，当 kubernetes 集群启用了 RBAC 的时候，将会出现权限错误。

------

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

```
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

Kubernetes applications can be executed via `spark-submit`. For example, to compute the value of pi, assuming the images
are set up as described above:

```
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar
```

The Spark master, specified either via passing the `--master` command line argument to `spark-submit` or by setting
`spark.master` in the application's configuration, must be a URL with the format `k8s://<api_server_url>`. Prefixing the
master string with `k8s://` will cause the Spark application to launch on the Kubernetes cluster, with the API server
being contacted at `api_server_url`. If no HTTP protocol is specified in the URL, it defaults to `https`. For example,
setting the master to `k8s://example.com:443` is equivalent to setting it to `k8s://https://example.com:443`, but to
connect without TLS on a different port, the master would be set to `k8s://http://example.com:8443`.

If you have a Kubernetes cluster setup, one way to discover the apiserver URL is by executing `kubectl cluster-info`.

```
> kubectl cluster-info
Kubernetes master is running at http://127.0.0.1:8080
```

In the above example, the specific Kubernetes cluster can be used with spark submit by specifying
`--master k8s://http://127.0.0.1:8080` as an argument to spark-submit.

Note that applications can currently only be executed in cluster mode, where the driver and its executors are running on
the cluster.

Finally, notice that in the above example we specify a jar with a specific URI with a scheme of `local://`. This URI is
the location of the example jar that is already in the Docker image. Using dependencies that are on your machine's local
disk is discussed below.

## Python 支持

With the ever growing support for Python by data scientists, we have supported the submission of PySpark applications.
These applications follow the general syntax that you would expect from other cluster managers. The submission of a PySpark
job is similar to the submission of Java/Scala applications except you do not supply a class, as expected. 
Here is how you would execute a Spark-Pi example:

```
bin/spark-submit \
  --deploy-mode cluster \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace <k8s-namespace> \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/driver-py:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/executor-py:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  --jars local:///opt/spark/examples/jars/spark-examples_2.11-2.1.0-k8s-0.3.1-SNAPSHOT.jar \
  local:///opt/spark/examples/src/main/python/pi.py 10
```

With Python support it is expected to distribute `.egg`, `.zip` and `.py` libraries to executors via the `--py-files` option. 
We support this as well, as seen with the following example:
​    

```
bin/spark-submit \
  --deploy-mode cluster \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace <k8s-namespace> \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/driver-py:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/executor-py:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  --jars local:///opt/spark/examples/jars/spark-examples_2.11-2.1.0-k8s-0.3.1-SNAPSHOT.jar \
  --py-files local:///opt/spark/examples/src/main/python/sort.py \
  local:///opt/spark/examples/src/main/python/pi.py 10
```

You may also customize your Docker images to use different `pip` packages that suit your use-case. As you can see
with the current `driver-py` Docker image we have commented out the current pip module support that you can uncomment
to use:

```
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

And bake into your docker image whichever PySpark files you wish to include by merely appending to the following exec
command with your appropriate file (i.e. MY_SPARK_FILE)

```
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

Application dependencies that are being submitted from your machine need to be sent to a **resource staging server** that the driver and executor can then communicate with to retrieve those dependencies. A YAML file denoting a minimal set of Kubernetes resources that runs this service is located in the file `conf/kubernetes-resource-staging-server.yaml`.
This YAML file configures a Deployment with one pod running the resource staging server configured with a ConfigMap, and exposes the server through a Service with a fixed NodePort. Deploying a resource staging server with the included YAML file requires you to have permissions to create Deployments, Services, and ConfigMaps.

To run the resource staging server with default configurations, the Kubernetes resources can be created:

```
kubectl create -f conf/kubernetes-resource-staging-server.yaml
```

and then you can compute the value of Pi as follows:

```
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.resourceStagingServer.uri=http://<address-of-any-cluster-node>:31000 \
  examples/jars/spark_examples_2.11-2.2.0.jar
```

The Docker image for the resource staging server may also be built from source, in a similar manner to the driver and executor images. The Dockerfile is provided in `dockerfiles/resource-staging-server/Dockerfile`.

The provided YAML file specifically sets the NodePort to 31000 on the service's specification. If port 31000 is not available on any of the nodes of your cluster, you should remove the NodePort field from the service's specification and allow the Kubernetes cluster to determine the NodePort itself. Be sure to provide the correct port in the resource staging server URI when submitting your application, in accordance to the NodePort chosen by the Kubernetes cluster.

### 不使用 Resource Staging Server 做依赖管理

Note that this resource staging server is only required for submitting local dependencies. If your application's dependencies are all hosted in remote locations like HDFS or http servers, they may be referred to by their appropriate remote URIs. Also, application dependencies can be pre-mounted into custom-built Docker images. Those dependencies can be added to the classpath by referencing them with `local://` URIs and/or setting the `SPARK_EXTRA_CLASSPATH` environment variable in your Dockerfiles.

### 访问 Kubernetes 集群

Spark-submit also supports submission through the [local kubectl proxy](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#using-kubectl-proxy). One can use the authenticating proxy to communicate with the api server directly without passing credentials to spark-submit.

The local proxy can be started by running:

```
kubectl proxy
```

If our local proxy were listening on port 8001, we would have our submission looking like the following:

```
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://http://127.0.0.1:8001 \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar
```

Communication between Spark and Kubernetes clusters is performed using the fabric8 kubernetes-client library.
The above mechanism using `kubectl proxy` can be used when we have authentication providers that the fabric8 kubernetes-client library does not support. Authentication using X509 Client Certs and OAuth tokens is currently supported.

## 动态 Executor Scale

Spark on Kubernetes supports Dynamic Allocation with cluster mode. This mode requires running
an external shuffle service. This is typically a [daemonset](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) with a provisioned [hostpath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) volume.
This shuffle service may be shared by executors belonging to different SparkJobs. Using Spark with dynamic allocation on Kubernetes assumes that a cluster administrator has set up one or more shuffle-service daemonsets in the cluster.

A sample configuration file is provided in `conf/kubernetes-shuffle-service.yaml` which can be customized as needed for a particular cluster. It is important to note that `spec.template.metadata.labels` are setup appropriately for the shuffle service because there may be multiple shuffle service instances running in a cluster. The labels give Spark applications a way to target a particular shuffle service.

For example, if the shuffle service we want to use is in the default namespace, and has pods with labels `app=spark-shuffle-service` and `spark-version=2.1.0`, we can use those tags to target that particular shuffle service at job launch time. In order to run a job with dynamic allocation enabled, the command may then look like the following:

```
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
  --conf spark.kubernetes.shuffle.labels="app=spark-shuffle-service,spark-version=2.1.0" \
  local:///opt/spark/examples/jars/spark_examples_2.11-2.2.0.jar 10 400000 2
```

## 高级

### 使用 TLS 加密 the Resource Staging Server

The default configuration of the resource staging server is not secured with TLS. It is highly recommended to configure this to protect the secrets and jars/files being submitted through the staging server.

The YAML file in `conf/kubernetes-resource-staging-server.yaml` includes a ConfigMap resource that holds the resource staging server's configuration. The properties can be adjusted here to make the resource staging server listen over TLS.
Refer to the [security](security.html) page for the available settings related to TLS. The namespace for the resource staging server is `kubernetes.resourceStagingServer`, so for example the path to the server's keyStore would be set by `spark.ssl.kubernetes.resourceStagingServer.keyStore`.

In addition to the settings specified by the previously linked security page, the resource staging server supports the following additional configurations:

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyPem</code></td>
  <td>(none)</td>
  <td>

```
Private key file encoded in PEM format that the resource staging server uses to secure connections over TLS. If this
is specified, the associated public key file must be specified in
<code>spark.ssl.kubernetes.resourceStagingServer.serverCertPem</code>. PEM files and a keyStore file (set by
<code>spark.ssl.kubernetes.resourceStagingServer.keyStore</code>) cannot both be specified at the same time.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.serverCertPem</code></td>
  <td>(none)</td>
  <td>

```
Certificate file encoded in PEM format that the resource staging server uses to secure connections over TLS. If this
is specified, the associated private key file must be specified in
<code>spark.ssl.kubernetes.resourceStagingServer.keyPem</code>. PEM files and a keyStore file (set by
<code>spark.ssl.kubernetes.resourceStagingServer.keyStore</code>) cannot both be specified at the same time.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyStorePasswordFile</code></td>
  <td>(none)</td>
  <td>

```
Provides the KeyStore password through a file in the container instead of a static value. This is useful if the
keyStore's password is to be mounted into the container with a secret.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.keyPasswordFile</code></td>
  <td>(none)</td>
  <td>

```
Provides the keyStore's key password using a file in the container instead of a static value. This is useful if the
keyStore's key password is to be mounted into the container with a secret.
```

  </td>
</tr>
</table>

Note that while the properties can be set in the ConfigMap, you will still need to consider the means of mounting the appropriate secret files into the resource staging server's container. A common mechanism that is used for this is to use [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) that are mounted as secret
volumes. Refer to the appropriate Kubernetes documentation for guidance and adjust the resource staging server's pecification in the provided YAML file accordingly.

Finally, when you submit your application, you must specify either a trustStore or a PEM-encoded certificate file to communicate with the resource staging server over TLS. The trustStore can be set with
`spark.ssl.kubernetes.resourceStagingServer.trustStore`, or a certificate file can be set with
`spark.ssl.kubernetes.resourceStagingServer.clientCertPem`. For example, our SparkPi example now looks like this:

```
bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkPi \
  --master k8s://https://<k8s-apiserver-host>:<k8s-apiserver-port> \
  --kubernetes-namespace default \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.1.0-kubernetes-0.3.1 \
  --conf spark.kubernetes.resourceStagingServer.uri=https://<address-of-any-cluster-node>:31000 \
  --conf spark.ssl.kubernetes.resourceStagingServer.enabled=true \
  --conf spark.ssl.kubernetes.resourceStagingServer.clientCertPem=/home/myuser/cert.pem \
  examples/jars/spark_examples_2.11-2.2.0.jar
```

### Spark Properties

Below are some other common properties that are specific to Kubernetes. Most of the other configurations are the same from the other deployment modes. See the [configuration page](configuration.html) for more information on those.

<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.kubernetes.namespace</code></td>
  <td><code>default</code></td>
  <td>

```
The namespace that will be used for running the driver and executor pods. When using
<code>spark-submit</code> in cluster mode, this can also be passed to <code>spark-submit</code> via the
<code>--kubernetes-namespace</code> command line argument.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.docker.image</code></td>
  <td><code>spark-driver:2.2.0</code></td>
  <td>

```
Docker image to use for the driver. Specify this using the standard
<a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> format.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.docker.image</code></td>
  <td><code>spark-executor:2.2.0</code></td>
  <td>

```
Docker image to use for the executors. Specify this using the standard
<a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> format.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.initcontainer.docker.image</code></td>
  <td><code>spark-init:2.2.0</code></td>
  <td>

```
Docker image to use for the init-container that is run before the driver and executor containers. Specify this using
the standard <a href="https://docs.docker.com/engine/reference/commandline/tag/">Docker tag</a> format. The
init-container is responsible for fetching application dependencies from both remote locations like HDFS or S3,
and from the resource staging server, if applicable.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.shuffle.namespace</code></td>
  <td><code>default</code></td>
  <td>

```
Namespace in which the shuffle service pods are present. The shuffle service must be
created in the cluster prior to attempts to use it.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.shuffle.labels</code></td>
  <td>(none)</td>
  <td>

```
Labels that will be used to look up shuffle service pods. This should be a comma-separated list of label key-value pairs,
where each label is in the format <code>key=value</code>. The labels chosen must be such that
they match exactly one shuffle service pod on each node that executors are launched.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.allocation.batch.size</code></td>
  <td><code>5</code></td>
  <td>

```
Number of pods to launch at once in each round of executor pod allocation.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.allocation.batch.delay</code></td>
  <td><code>1</code></td>
  <td>

```
Number of seconds to wait between each round of executor pod allocation.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.caCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the CA cert file for connecting to the Kubernetes API server over TLS when starting the driver. This file
must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide
a scheme).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client key file for authenticating against the Kubernetes API server when starting the driver. This file
must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not provide
a scheme).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.clientCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client cert file for authenticating against the Kubernetes API server when starting the driver. This
file must be located on the submitting machine's disk. Specify this as a path as opposed to a URI (i.e. do not
provide a scheme).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.submission.oauthToken</code></td>
  <td>(none)</td>
  <td>

```
OAuth token to use when authenticating against the Kubernetes API server when starting the driver. Note
that unlike the other authentication options, this is expected to be the exact string value of the token to use for
the authentication.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.caCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the CA cert file for connecting to the Kubernetes API server over TLS from the driver pod when requesting
executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod.
Specify this as a path as opposed to a URI (i.e. do not provide a scheme).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client key file for authenticating against the Kubernetes API server from the driver pod when requesting
executors. This file must be located on the submitting machine's disk, and will be uploaded to the driver pod.
Specify this as a path as opposed to a URI (i.e. do not provide a scheme). If this is specified, it is highly
recommended to set up TLS for the driver submission server, as this value is sensitive information that would be
passed to the driver pod in plaintext otherwise.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.clientCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client cert file for authenticating against the Kubernetes API server from the driver pod when
requesting executors. This file must be located on the submitting machine's disk, and will be uploaded to the
driver pod. Specify this as a path as opposed to a URI (i.e. do not provide a scheme).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.oauthToken</code></td>
  <td>(none)</td>
  <td>

```
OAuth token to use when authenticating against the against the Kubernetes API server from the driver pod when
requesting executors. Note that unlike the other authentication options, this must be the exact string value of
the token to use for the authentication. This token value is uploaded to the driver pod. If this is specified, it is
highly recommended to set up TLS for the driver submission server, as this value is sensitive information that would
be passed to the driver pod in plaintext otherwise.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.driver.serviceAccountName</code></td>
  <td><code>default</code></td>
  <td>

```
Service account that is used when running the driver pod. The driver pod uses this service account when requesting
executor pods from the API server. Note that this cannot be specified alongside a CA cert file, client key file,
client cert file, and/or OAuth token.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.caCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the CA cert file for connecting to the Kubernetes API server over TLS from the resource staging server when
it monitors objects in determining when to clean up resource bundles.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.clientKeyFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client key file for authenticating against the Kubernetes API server from the resource staging server
when it monitors objects in determining when to clean up resource bundles. The resource staging server must have
credentials that allow it to view API objects in any namespace.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.clientCertFile</code></td>
  <td>(none)</td>
  <td>

```
Path to the client cert file for authenticating against the Kubernetes API server from the resource staging server
when it monitors objects in determining when to clean up resource bundles. The resource staging server must have
credentials that allow it to view API objects in any namespace.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.oauthToken</code></td>
  <td>(none)</td>
  <td>

```
OAuth token value for authenticating against the Kubernetes API server from the resource staging server
when it monitors objects in determining when to clean up resource bundles. The resource staging server must have
credentials that allow it to view API objects in any namespace. Note that this cannot be set at the same time as
<code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code>.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code></td>
  <td>(none)</td>
  <td>

```
File containing the OAuth token to use when authenticating against the against the Kubernetes API server from the
resource staging server, when it monitors objects in determining when to clean up resource bundles. The resource
staging server must have credentials that allow it to view API objects in any namespace. Note that this cannot be
set at the same time as <code>spark.kubernetes.authenticate.resourceStagingServer.oauthToken</code>.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.authenticate.resourceStagingServer.useServiceAccountCredentials</code></td>
  <td>true</td>
  <td>

```
Whether or not to use a service account token and a service account CA certificate when the resource staging server
authenticates to Kubernetes. If this is set, interactions with Kubernetes will authenticate using a token located at
<code>/var/run/secrets/kubernetes.io/serviceaccount/token</code> and the CA certificate located at
<code>/var/run/secrets/kubernetes.io/serviceaccount/ca.crt</code>. Note that if
<code>spark.kubernetes.authenticate.resourceStagingServer.oauthTokenFile</code> is set, it takes precedence
over the usage of the service account token file. Also, if
<code>spark.kubernetes.authenticate.resourceStagingServer.caCertFile</code> is set, it takes precedence over using
the service account's CA certificate file. This generally should be set to true (the default value) when the
resource staging server is deployed as a Kubernetes pod, but should be set to false if the resource staging server
is deployed by other means (i.e. when running the staging server process outside of Kubernetes). The resource
staging server must have credentials that allow it to view API objects in any namespace.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.memoryOverhead</code></td>
  <td>executorMemory * 0.10, with minimum of 384</td>
  <td>

```
The amount of off-heap memory (in megabytes) to be allocated per executor. This is memory that accounts for things
like VM overheads, interned strings, other native overheads, etc. This tends to grow with the executor size
(typically 6-10%).
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.labels</code></td>
  <td>(none)</td>
  <td>

```
Custom labels that will be added to the driver pod. This should be a comma-separated list of label key-value pairs,
where each label is in the format <code>key=value</code>. Note that Spark also adds its own labels to the driver pod
for bookkeeping purposes.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.annotations</code></td>
  <td>(none)</td>
  <td>

```
Custom annotations that will be added to the driver pod. This should be a comma-separated list of label key-value
pairs, where each annotation is in the format <code>key=value</code>.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.labels</code></td>
  <td>(none)</td>
  <td>

```
Custom labels that will be added to the executor pods. This should be a comma-separated list of label key-value
pairs, where each label is in the format <code>key=value</code>. Note that Spark also adds its own labels to the
executor pods for bookkeeping purposes.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.executor.annotations</code></td>
  <td>(none)</td>
  <td>

```
Custom annotations that will be added to the executor pods. This should be a comma-separated list of annotation
key-value pairs, where each annotation is in the format <code>key=value</code>.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.driver.pod.name</code></td>
  <td>(none)</td>
  <td>

```
Name of the driver pod. If not set, the driver pod name is set to "spark.app.name" suffixed by the current timestamp
to avoid name conflicts.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.submission.waitAppCompletion</code></td>
  <td><code>true</code></td>
  <td>

```
In cluster mode, whether to wait for the application to finish before exiting the launcher process.  When changed to
false, the launcher has a "fire-and-forget" behavior when launching the Spark job.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.port</code></td>
  <td><code>10000</code></td>
  <td>

```
Port for the resource staging server to listen on when it is deployed.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.uri</code></td>
  <td>(none)</td>
  <td>

```
URI of the resource staging server that Spark should use to distribute the application's local dependencies. Note
that by default, this URI must be reachable by both the submitting machine and the pods running in the cluster. If
one URI is not simultaneously reachable both by the submitter and the driver/executor pods, configure the pods to
access the staging server at a different URI by setting
<code>spark.kubernetes.resourceStagingServer.internal.uri</code> as discussed below.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.resourceStagingServer.internal.uri</code></td>
  <td>Value of <code>spark.kubernetes.resourceStagingServer.uri</code></td>
  <td>

```
URI of the resource staging server to communicate with when init-containers bootstrap the driver and executor pods
with submitted local dependencies. Note that this URI must by the pods running in the cluster. This is useful to
set if the resource staging server has a separate "internal" URI that must be accessed by components running in the
cluster.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStore</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.trustStore</code></td>
  <td>

```
Location of the trustStore file to use when communicating with the resource staging server over TLS, as
init-containers bootstrap the driver and executor pods with submitted local dependencies. This can be a URI with a
scheme of <code>local://</code>, which denotes that the file is pre-mounted on the pod's disk. A uri without a
scheme or a scheme of <code>file://</code> will result in this file being mounted from the submitting machine's
disk as a secret into the init-containers.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStorePassword</code></td>
  <td>Value of <code><code>spark.ssl.kubernetes.resourceStagingServer.trustStorePassword</code></td>
  <td>

```
Password of the trustStore file that is used when communicating with the resource staging server over TLS, as
init-containers bootstrap the driver and executor pods with submitted local dependencies.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.trustStoreType</code></td>
  <td>Value of <code><code>spark.ssl.kubernetes.resourceStagingServer.trustStoreType</code></td>
  <td>

```
Type of the trustStore file that is used when communicating with the resource staging server over TLS, when
init-containers bootstrap the driver and executor pods with submitted local dependencies.
```

  </td>
</tr>
<tr>
  <td><code>spark.ssl.kubernetes.resourceStagingServer.internal.clientCertPem</code></td>
  <td>Value of <code>spark.ssl.kubernetes.resourceStagingServer.clientCertPem</code></td>
  <td>

```
Location of the certificate file to use when communicating with the resource staging server over TLS, as
init-containers bootstrap the driver and executor pods with submitted local dependencies. This can be a URI with a
scheme of <code>local://</code>, which denotes that the file is pre-mounted on the pod's disk. A uri without a
scheme or a scheme of <code>file://</code> will result in this file being mounted from the submitting machine's
disk as a secret into the init-containers.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.mountdependencies.jarsDownloadDir</code></td>
  <td><code>/var/spark-data/spark-jars</code></td>
  <td>

```
Location to download jars to in the driver and executors. This will be mounted as an empty directory volume
into the driver and executor containers.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.mountdependencies.filesDownloadDir</code></td>
  <td><code>/var/spark-data/spark-files</code></td>
  <td>

```
Location to download files to in the driver and executors. This will be mounted as an empty directory volume
into the driver and executor containers.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.report.interval</code></td>
  <td><code>1s</code></td>
  <td>

```
Interval between reports of the current Spark job status in cluster mode.
```

  </td>
</tr>
<tr>
  <td><code>spark.kubernetes.docker.image.pullPolicy</code></td>
  <td><code>IfNotPresent</code></td>
  <td>

```
Docker image pull policy used when pulling Docker images with Kubernetes.
```

  </td>
</tr>
<tr>
   <td><code>spark.kubernetes.driver.limit.cores</code></td>
   <td>(none)</td>
   <td>

```
 Specify the hard cpu limit for the driver pod
```

   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.executor.limit.cores</code></td>
   <td>(none)</td>
   <td>

```
 Specify the hard cpu limit for a single executor pod
```

   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.node.selector.[labelKey]</code></td> 
   <td>(none)</td>
   <td>

```
 Adds to the node selector of the driver pod and executor pods, with key <code>labelKey</code> and the value as the 
 configuration's value. For example, setting <code>spark.kubernetes.node.selector.identifier</code> to <code>myIdentifier</code>
 will result in the driver pod and executors having a node selector with key <code>identifier</code> and value 
  <code>myIdentifier</code>. Multiple node selector keys can be added by setting multiple configurations with this prefix.
</td>
```

  </tr>
 <tr>
   <td><code>spark.executorEnv.[EnvironmentVariableName]</code></td> 
   <td>(none)</td>
   <td>

```
 Add the environment variable specified by <code>EnvironmentVariableName</code> to
 the Executor process. The user can specify multiple of these to set multiple environment variables.
```

   </td>
 </tr>
 <tr>
   <td><code>spark.kubernetes.driverEnv.[EnvironmentVariableName]</code></td> 
   <td>(none)</td>
   <td>

```
 Add the environment variable specified by <code>EnvironmentVariableName</code> to
 the Driver process. The user can specify multiple of these to set multiple environment variables.
```

   </td>
 </tr>
</table>

## 目前的限制

在 Kubernetes 上运行 spark 还处于实验状态。当前的实现中还有一些限制，可能会在未来解决：

- 应用程序只能在 cluster mode 下运行。
- 只支持 Scala、Java 和 Python 应用。