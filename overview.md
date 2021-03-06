# `Spark`概述

`Apache Spark`是一个快速的，多用途的集群计算系统。它提供了`Java`，`Scala`，`Python`和`R`的高级`API`，以及一个支持通用的执行图计算的优化过的引擎。它还支持一组丰富的高级工具，包括使用`SQL`处理结构化数据处理的 [`Spark SQL`](https://spark.apache.org/docs/latest/sql-programming-guide.html)，用于机器学习的 [`MLlib`](https://spark.apache.org/docs/latest/ml-guide.html)，用于图计算的 [`GraphX`](https://spark.apache.org/docs/latest/graphx-programming-guide.html)，以及 [`Spark Streaming`](https://spark.apache.org/docs/latest/streaming-programming-guide.html)。

# 安全

`Spark中`的安全性默认为`OFF`。这可能意味着您很容易受到默认攻击。在下载和运行`Spark`之前，请参阅[`Spark Security`](https://spark.apache.org/docs/latest/security.html)。

# 下载

从该项目官网的 [下载页面](http://spark.apache.org/downloads.html) 获取`Spark`。该文档用于`Spark 2.4.5`版本。`Spark`可以通过`Hadoop client`库使用`HDFS`和`YARN`。下载一个预编译主流`Hadoop`版本比较麻烦。用户可以下载"Hadoop free"的二进制，并且可以通过[设置`Spark`的`classpath`](https://spark.apache.org/docs/latest/hadoop-provided.html) 来与任何的`Hadoop`版本一起运行`Spark`。`Scala`和`Java`用户可以在他们的工程中通过`Maven`的方式引入`Spark`，并且在将来`Python`用户也可以从`PyPI`中安装`Spark`。

如果您希望从源码中编译一个`Spark`，请访问[编译`Spark`](https://spark.apache.org/docs/latest/building-spark.html)。

`Spark`可以在`windows`和`unix`类似的系统（例如，`Linux`，`Mac OS`）上运行。它可以很容易的在一台本地机器上运行 -你只需要安装一个`JAVA`环境并配置`PATH`环境变量，或者让`JAVA_HOME`指向你的`JAVA`安装路径

`Spark`可运行在`Java 8+`，`Python 2.7+/3.4+` 和`R 3.1+`的环境上。针对`Scala API`，`Spark 2.4.5`使用了`Scala 2.12`。您将需要去使用一个可兼容的`Scala`版本（`2.12.x`）。

请注意，从`Spark 2.2.0`起，对`Java 7`，`Python 2.6`和旧的`Hadoop 2.6.5`之前版本的支持均已被删除。自`2.3.0`起，对`Scala 2.10`的支持被删除。 自`Spark 2.4.1`起，对`Scala 2.11`的支持已被弃用，将在`Spark 3.0`中删除。

# 运行示例和`Shell`

`Spark`自带了几个示例程序。`Scala`，`Java`，`Python`和`R`示例在 `examples/src/main`目录中。要运行`Java`或`Scala`中的某个示例程序，在最顶层的`Spark`目录中使用 `bin/run-example <class> [params]` 命令即可。（这个命令底层调用了 [`spark-submit` 脚本](https://spark.apache.org/docs/latest/submitting-applications.html)去加载应用程序）。例如，

```bash
./bin/run-example SparkPi 10 
```

您也可以通过一个改进版的`Scala shell`来运行交互式的`Spark`。这是一个来学习该框架比较好的方式。

```bash
./bin/spark-shell --master local[2] 
```

该 `--master` 选项可以指定为[针对分布式集群的`master URL`](https://spark.apache.org/docs/latest/submitting-applications.html#master-urls)，或者 以 `local`模式使用1个线程在本地运行，`local[N]` 会使用`N`个线程在本地运行。你应该先使用`local`模式进行测试。可以通过`–help`指令来获取`spark-shell`的所有配置项。`Spark`同样支持`Python API`。在`Python interpreter`（解释器）中运行交互式的`Spark`，请使用 `bin/pyspark`:

```bash
./bin/pyspark --master local[2] 
```

`Python`中也提供了应用示例。例如，

```bash
./bin/spark-submit examples/src/main/python/pi.py 10 
```

从`1.4`开始（仅包含了`DataFrames APIs`）`Spark`也提供了一个用于实验性的[`R API`](https://spark.apache.org/docs/latest/sparkr.html)。为了在`R interpreter`（解释器）中运行交互式的`Spark`，请执行 `bin/sparkR`:

```bash
./bin/sparkR --master local[2] 
```

`R`中也提供了应用示例。例如，

```bash
./bin/spark-submit examples/src/main/r/dataframe.R 
```

# 在集群上运行

该`Spark`[集群模式概述](https://spark.apache.org/docs/latest/cluster-overview.html) 说明了在集群上运行的主要的概念。`Spark`既可以独立运行，也可以在一些现有的`Cluster Manager`（集群管理器）上运行。它当前提供了几种用于部署的选项:

*   [`Standalone Deploy Mode`](https://spark.apache.org/docs/latest/spark-standalone.html)：在私有集群上部署`Spark`最简单的方式
*   [`Apache Mesos`](https://spark.apache.org/docs/latest/running-on-mesos.html)
*   [`Hadoop YARN`](https://spark.apache.org/docs/latest/running-on-yarn.html)
*   [`Kubernetes`](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
