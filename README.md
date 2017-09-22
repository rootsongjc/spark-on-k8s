## Spark on Kubernetes
Spark on Kubernetes 中文文档 https://jimmysong.io/spark-on-k8s/

**本文档使用 Jekyll 构建**

### 文档构建指南

#### 环境要求
在 `spark-on-k8s`  目录下执行：

```bash
$ gem install bundle
$ bundle install 
```

软件依赖通过 `Gemfile` 和 `Gemfile.lock` 声明，使用 bundler 管理，该命令将安装 jekyll 和编译时需要用到的插件。

#### 生成文档

```bash
$ jekyll build
```
## 启动Server

```bash
$ jekyll server
```

#### 增加新文档

直接在根目录添加 markdown 文件，请 metadata 格式要求。

主页的文档是 `index.md`。