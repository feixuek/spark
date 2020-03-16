# `RDD`编程指南

*   [概述](#概述)
*   [与Spark链接](#与Spark链接)
*   [初始化Spark](#初始化Spark)
    *   [使用Shell](#使用Shell)
*   [弹性分布式数据集](#弹性分布式数据集)
    *   [并行集合](#并行集合)
    *   [外部数据集）](#外部数据集)
    *   [RDD操作](#RDD操作)
        *   [基础](#基础)
        *   [传递函数给Spark](#传递函数给Spark)
        *   [理解闭包](#理解闭包)
            *   [示例](#示例)
            *   [本地vs集群模式](#本地vs集群模式)
            *   [打印RDD的元素](#打印RDD的元素)
        *   [与键值对一起使用](#与键值对一起使用)
        *   [Transformations](#Transformations)
        *   [Actions](#Actions)
        *   [Shuffle](#Shuffle)
            *   [幕后）](#幕后)
            *   [性能影响](#性能影响)
    *   [RDD持久化）](#RDD持久化)
        *   [如何选择存储级别 ](#如何选择存储级别)
        *   [删除数据](#删除数据)
*   [共享变量](#共享变量)
    *   [广播变量](#广播变量)
    *   [累加器](#累加器)
*   [部署应用到集群中](#部署应用到集群中)
*   [从Java/Scala启动Spark-jobs](#从Java/Scala启动Spark-jobs)
*   [单元测试](#单元测试)
*   [快速链接](#快速链接)

## 概述

在一个较高的概念上来说，每一个`Spark`应用程序由一个在集群上运行着用户的 `main` 函数和执行各种并行操作的组成。`Spark`提供的主要抽象是一个 _弹性分布式数据集_（`RDD`），它是可以执行并行操作且跨集群节点的元素的集合。`RDD`可以从一个`Hadoop`文件系统（或者任何其它`Hadoop` 支持的文件系统），或者一个在驱动程序中已存在的`Scala`集合，以及通过`transforming`来创建。用户为了让它在整个并行操作中更高效的重用，也许会让`Spark persist`一个`RDD`到内存中。最后,`RDD`会自动的从节点故障中恢复。

在`Spark`中的第二个抽象是能够用于并行操作的 _shared variables_（共享变量），默认情况下，当`Spark`的一个函数作为一组不同节点上的任务运行时，它将每一个变量的副本应用到每一个任务的函数中去。有时候，一个变量需要在整个任务中，或者在任务和驱动程序之间来共享。`Spark`支持两种类型的共享变量：_broadcast variables_（广播变量），它可以用于在所有节点上缓存一个值，和 _accumulators_（累加器），他是一个只能被 `added` 的变量，例如`counters`和`sums`。

本指南介绍了每一种`Spark`所支持的语言的特性。如果您启动`Spark`的交互式`shell` - 针对`Scala shell`使用 `bin/spark-shell` 或者针对`Python`使用`bin/pyspark` 是很容易来学习的。

## 与Spark链接
`Spark 2.4.5`适用于`Python 2.7+`或`Python 3.4+`。它可以使用标准的`CPython`解释器，因此可以使用`NumPy`之类的`C`库。它还适用于`PyPy 2.3+`。

在`Spark 2.2.0`中删除了对`Python 2.6`的支持。

`Python`中的`Spark`应用程序既可以在运行时与`bin/spark-submit`包含`Spark`的脚本一起运行，也可以将其包含在`setup.py`中，如下所示：
```bash
    install_requires=[
        'pyspark=={site.SPARK_VERSION}'
    ]
```
要在`Python`中运行`Spark`应用程序而无需`pip`安装`PySpark`，请使用`bin/spark-submitSpark`目录中的脚本。该脚本将加载`Spark`的`Java/Scala`库，并允许您将应用程序提交到集群。您还可以用`bin/pyspark`来启动交互式`Python Shell`。

如果您想访问`HDFS`数据，则需要使用`PySpark`的构建链接到您的`HDFS`版本。 对于常用的`HDFS`版本，`Spark`主页上也提供了预构建的软件包。

最后，您需要将一些`Spark`类导入到程序中。添加以下行：
```Python
from pyspark import SparkContext, SparkConf
```
`PySpark`在驱动程序和工作程序中都需要相同版本的`Python`。它在`PATH`中使用默认的`python`版本，您可以通过指定要使用的`Python`版本`PYSPARK_PYTHON`，例如：
```bash
$ PYSPARK_PYTHON=python3.4 bin/pyspark
$ PYSPARK_PYTHON=/opt/pypy-2.5/bin/pypy bin/spark-submit examples/src/main/python/pi.py
```

## 初始化Spark

`Spark`程序必须做的第一件事是创建一个[`SparkContext`](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.SparkContext)对象，它告诉`Spark`如何访问集群。 要创建`SparkContext`，首先需要构建一个包含有关应用程序信息的[`SparkConf`](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.SparkConf)对象。
```Python
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```

这个 `appName` 参数是一个在集群`UI`上展示应用程序的名称。`master` 是一个[`Spark`，`Mesos`或`YARN`的`cluster URL`](https://spark.apache.org/docs/latest/submitting-applications.html)，或者指定为在`local mode`（本地模式）中运行的 "local" 字符串。在实际工作中，当在集群上运行时，您不希望在程序中将`master` 给硬编码，而是用 [使用 `spark-submit` 启动应用](https://spark.apache.org/docs/latest/submitting-applications.html) 并且接收它。然而，对于本地测试和单元测试，您可以通过 "local" 来运行`Spark`进程。

### 使用Shell

在`PySpark shell`中，已经为你创建了一个特殊的解释器`SparkContext`，名为`sc`。 制作自己的`SparkContext`将无法正常工作。 您可以使用`--master`参数设置上下文连接到的主服务器，并且可以通过将逗号分隔的列表传递给`--py-files`将`Python` `.zip`，`.egg`或`.py`文件添加到运行时路径。 您还可以通过向`--packages`参数提供以逗号分隔的`Maven`坐标列表，将依赖项（例如`Spark`包）添加到`shell`会话中。 任何可能存在依赖关系的其他存储库（例如`Sonatype`）都可以传递给`--repositories`参数。 必要时，必须使用`pip`手动安装`Spark`软件包具有的任何`Python`依赖项（在该软件包的`requirements.txt`中列出）。 例如，要在四个核心上运行`bin/pyspark`，请使用：

```bash
$ ./bin/pyspark --master local[4]
```

或者，要将`code.py`添加到搜索路径（以便以后能够`import code`），请使用：

```bash
$ ./bin/pyspark --master local[4] --py-files code.py
```

有关选项的完整列表，请运行`pyspark --help`。 在幕后，`pyspark`调用更一般的[`spark-submit`脚本](https://spark.apache.org/docs/latest/submitting-applications.html)。

也可以在增强的`Python`解释器`IPython中`启动`PySpark shell`。 `PySpark`适用于`IPython 1.0.0`及更高版本。 要使用`IPython`，请在运行`bin/pyspark`时将`PYSPARK_DRIVER_PYTHON`变量设置为`ipython`：

```bash
$ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark
```

要使用`Jupyter notebook`(以前被称为`IPython notebook`),

```bash
$ PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS=notebook ./bin/pyspark
```

您可以通过设置`PYSPARK_DRIVER_PYTHON_OPTS`来自定义`ipython`或`jupyter`命令。

启动`Jupyter Notebook`服务器后，您可以从"文件"选项卡创建一个新的"Python 2"笔记本。 在`notebook`内部，您可以在开始尝试使用`Jupyter notebook`中的`Spark`之前输入命令`%pylab inline`作为`notebook`的一部分。

## 弹性分布式数据集

`Spark`主要以一个 _弹性分布式数据集_（`RDD`）的概念为中心，它是一个容错且可以执行并行操作的元素的集合。有两种方法可以创建`RDD`：在你的驱动程序中 _parallelizing_ 一个已存在的集合，或者在外部存储系统中引用一个数据集，例如，一个共享文件系统，`HDFS`，`HBase`，或者提供`Hadoop InputFormat` 的任何数据源。

### 并行集合

通过在驱动程序中的现有可迭代或集合上调用`SparkContext`的`parallelize`方法来创建并行化集合。 复制集合的元素以形成可以并行操作的分布式数据集。 例如，以下是如何创建包含数字1到5的并行化集合：

```Python
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
```

一旦创建，分布式数据集（`distData`）可以并行操作。 例如，我们可以调用`distData.reduce(lambda a,b：a + b)`来添加列表的元素。 我们稍后将描述对分布式数据集的操作。

并行集合中一个很重要参数是 _partitions_（分区）的数量，它可用来切割`dataset`。`Spark`将在集群中的每一个分区上运行一个任务。通常您希望群集中的每一个`CPU`计算2-4个分区。一般情况下，`Spark`会尝试根据您的群集情况来自动的设置的分区的数量。当然，您也可以将分区数作为第二个参数传递到 `parallelize`（例如，`sc.parallelize(data, 10)`）方法中来手动的设置它。注意：代码中的一些地方会使用术语片（分区的同义词）以保持向后兼容.

### 外部数据集

`PySpark`可以从`Hadoop`支持的任何存储源创建分布式数据集，包括本地文件系统, `HDFS`, `Cassandra`, `HBase`, [`Amazon S3`](http://wiki.apache.org/hadoop/AmazonS3)等。 `Spark`支持文本文件, [`SequenceFiles`](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html), 和任何其他`Hadoop`[`InputFormat`](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html).

可以使用`SparkContext`的`textFile`方法创建文本文件`RDD`。 此方法获取文件的`URI`（计算机上的本地路径，或`hdfs//`，`s3a//`等`URI`）并将其作为行集合读取。 这是一个示例调用：

```Python
>>> distFile = sc.textFile("data.txt")
```

创建后，`distFile`可以由数据集操作执行。 例如，我们可以使用`map`和`reduce`计算所有行的大小: `distFile.map(lambda s: len(s)).reduce(lambda a, b: a + b)`.

有关使用`Spark`读取文件的一些注意事项:

*   如果在本地文件系统上使用路径，则还必须可以在工作节点上的相同路径上访问该文件。 将文件复制到所有工作者或使用网络安装的共享文件系统。

*   `Spark`的所有基于文件的输入方法（包括`textFile`）都支持在目录，压缩文件和通配符上运行。 例如，您可以使用 `textFile("/my/directory")`, `textFile("/my/directory/*.txt")`, and `textFile("/my/directory/*.gz")`.

*   `textFile`方法还采用可选的第二个参数来控制文件的分区数。 默认情况下，`Spark`为文件的每个块创建一个分区（`HDFS`中默认为128MB），但您也可以通过传递更大的值来请求更多的分区。 请注意，您不能拥有比块少的分区。

除文本文件外，`Spark`的`Python API`还支持其他几种数据格式：

*   `SparkContext.wholeTextFiles` 允许您读取包含多个小文本文件的目录，并将它们作为（文件名，内容）对返回。 这与`textFile`形成对比，`textFile`将在每个文件中每行返回一条记录。

*   `RDD.saveAsPickleFile` 和 `SparkContext.pickleFile` 支持以包含`pickle Python`对象的简单格式保存`RDD`。 批处理用于`pickle`序列化，默认批处理大小为10。

*   `SequenceFile`和`Hadoop`输入/输出格式

**注意** 此功能目前标记为`Experimental`，适用于高级用户。 将来可能会使用基于`Spark SQL`的读/写支持替换它，在这种情况下，`Spark SQL`是首选方法。

**可写支持**

`PySpark SequenceFile`支持在`Java中`加载键值对的`RDD`，将`Writable`转换为基本`Java`类型，并使用[`Pyrolite`](https://github.com/irmen/Pyrolite/)挖掘生成的`Java`对象。将键值对的`RDD`保存到`SequenceFile`时，`PySpark`会反过来。 它将`Python`对象解开为`Java`对象，然后将它们转换为`Writable`。 以下`Writable`会自动转换：

| Writable Type | Python Type |
| --- | --- |
| Text | unicode str |
| IntWritable | int |
| FloatWritable | float |
| DoubleWritable | float |
| BooleanWritable | bool |
| BytesWritable | bytearray |
| NullWritable | None |
| MapWritable | dict |

数组不是开箱即用的。 用户在读取或写入时需要指定自定义`ArrayWritable`子类型。 编写时，用户还需要指定将数组转换为自定义`ArrayWritable`子类型的自定义转换器。 在读取时，默认转换器将自定义`ArrayWritable`子类型转换为`Java Object[]`，然后将其`pickle`到`Python`元组。 要为原始类型的数组获取`Python` `array.array`，用户需要指定自定义转换器。

**保存和加载SequenceFiles**

与文本文件类似，可以通过指定路径来保存和加载`SequenceFiles`。 可以指定键和值类，但对于标准`Writable`，这不是必需的。

```Python
>>> rdd = sc.parallelize(range(1, 4)).map(lambda x: (x, "a" * x))
>>> rdd.saveAsSequenceFile("path/to/file")
>>> sorted(sc.sequenceFile("path/to/file").collect())
[(1, u'a'), (2, u'aa'), (3, u'aaa')]
```

**保存和加载其他Hadoop输入/输出格式**

`PySpark`还可以为"新"和"旧"`Hadoop MapReduce API`读取任何`Hadoop InputFormat`或编写任何`Hadoop OutputFormat`。 如果需要，`Hadoop`配置可以作为`Python dict`传入。 以下是使用`Elasticsearch ESInputFormat`的示例：

```Python
$ ./bin/pyspark --jars /path/to/elasticsearch-hadoop.jar
>>> conf = {"es.resource" : "index/type"}  # assume Elasticsearch is running on localhost defaults
>>> rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",
                             "org.apache.hadoop.io.NullWritable",
                             "org.elasticsearch.hadoop.mr.LinkedMapWritable",
                             conf=conf)
>>> rdd.first()  # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
```

请注意，如果`InputFormat`仅依赖于`Hadoop`配置和/或输入路径，并且可以根据上表轻松转换键和值类，则此方法应适用于此类情况。

如果您有自定义序列化二进制数据（例如从`Cassandra/HBase`加载数据），那么您首先需要将`Scala/Java`端的数据转换为可由`Pyrolite`的`pickler`处理的数据。 为此提供了[`Converter`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.api.python.Converter)特性。 只需扩展此特征并在`convert`方法中实现转换代码。 请记住确保将此类以及访问`InputFormat`所需的任何依赖项打包到`Spark`作业`jar中`并包含在`PySpark`类路径中。

有关使用`Cassandra/HBase InputFormat`和`OutputFormat`以及自定义转换器的示例，请参阅[`Python`示例](https://github.com/apache/spark/tree/master/examples/src/main/python)和[`Converter`示例](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/pythonconverters)。

### RDD操作

`RDD` 支持两种类型的操作：_transformations_：它会在一个已存在的`dataset`上创建一个新的`dataset`，和 _actions_：将在`dataset` 上运行的计算后返回到驱动程序。例如，`map` 是一个通过让每个数据集元素都执行一个函数，并返回的新`RDD`结果的`transformation`，`reduce` 通过执行一些函数，聚合`RDD`中所有元素，并将最终结果给返回驱动程序（虽然也有一个并行 `reduceByKey` 返回一个分布式数据集）的`action`.

`Spark`中所有的`transformations`都是 _lazy_，因此它不会立刻计算出结果。相反，他们只记得应用于一些基本数据集的转换（例如. 文件）。只有当需要返回结果给驱动程序时，`transformations`才开始计算。这种设计使`Spark`的运行更高效。例如，我们可以了解到，`map` 所创建的数据集将被用在`reduce` 中，并且只有`reduce` 的计算结果返回给驱动程序，而不是映射一个更大的数据集.

默认情况下，每次你在`RDD`运行一个`action`时，每个`transformed RDD`都会被重新计算。但是，您也可用 `persist`（或 `cache`）方法将`RDD`持久化到内存中；在这种情况下，`Spark`为了下次查询时可以更快地访问，会把数据保存在集群上。此外，还支持持续持久化`RDD`到磁盘，或复制到多个结点。

#### 基础

为了说明`RDD`基础知识，请考虑以下简单程序：

```Python
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
```

第一行定义来自外部文件的基本`RDD`。 此`dataset`未加载到内存中或以其他方式执行：`lines`仅仅是指向文件的指针。 第二行将`lineLengths`定义为`map`转换的结果。 同样，由于懒惰，`lineLengths`不会立即计算。 最后，我们运行`reduce`，这是一个动作。 此时，`Spark`将计算分解为在不同机器上运行的任务，并且每台机器都运行其部分映射和本地缩减，仅返回其对驱动程序的答案。

如果我们以后想再次使用`lineLengths`，我们可以添加：

```Python
lineLengths.persist()
```

在`reduce`之前，这将导致`lineLengths`在第一次计算之后保存在内存中。

#### 传递函数给Spark

`Spark`的`API`在很大程度上依赖于在驱动程序中传递函数以在集群上运行。 有三种建议的方法可以做到这一点：

*   [`Lambda`表达式](https://docs.python.org/2/tutorial/controlflow.html#lambda-expressions), 对于可以作为表达式编写的简单函数。 （`Lambdas`不支持多语句函数或不返回值的语句。）
*   调用`Spark`的函数内部的本地`def`，用于更长的代码。
*   模块中的顶级函数。

例如，要传递比使用`lambda`支持的更长的函数，请考虑以下代码：

```Python
"""MyScript.py"""
if __name__ == "__main__":
    def myFunc(s):
        words = s.split(" ")
        return len(words)

    sc = SparkContext(...)
    sc.textFile("file.txt").map(myFunc)
```

请注意，虽然也可以将引用传递给类实例中的方法（而不是单例对象），但这需要发送包含该类的对象以及方法。 例如，考虑：

```Python
class MyClass(object):
    def func(self, s):
        return s
    def doStuff(self, rdd):
        return rdd.map(self.func)
```

在这里，如果我们创建一个新的`MyClass`并在其上调用`doStuff`，那里的`map`会引用该`MyClass`实例的`func`方法，因此需要将整个对象发送到集群。

以类似的方式，访问外部对象的字段将引用整个对象：

```Python
class MyClass(object):
    def __init__(self):
        self.field = "Hello"
    def doStuff(self, rdd):
        return rdd.map(lambda s: self.field + s)
```

要避免此问题，最简单的方法是将字段复制到局部变量中，而不是从外部访问它：

```Python
def doStuff(self, rdd):
    field = self.field
    return rdd.map(lambda s: field + s)
```

#### 理解闭包

`Spark`的一个难点是在跨集群执行代码时理解变量和方法的范围和生命周期。 修改其范围之外的变量的`RDD`操作可能经常引起混淆。 在下面的示例中，我们将查看使用`foreach()`递增计数器的代码，但同样的问题也可能发生在其他操作中。

##### 示例

考虑一个简单的`RDD`元素求和，以下行为可能不同，具体取决于是否在同一个`JVM`中执行。一个常见的例子是当`Spark`运行在 `local` 本地模式（`--master = local[n]`）时，与部署`Spark`应用到群集（例如，通过`spark-submit`到`YARN`）:

```Python
counter = 0
rdd = sc.parallelize(data)

# Wrong: Don't do this!!
def increment_counter(x):
    global counter
    counter += x
rdd.foreach(increment_counter)

print("Counter value: ", counter)
```

##### 本地vs集群模式

上面的代码行为未定义，可能无法按预期工作。为了执行作业时，`Spark`将`RDD`操作分解为`task`，每个`task`由`executor`执行。在执行之前，`Spark`计算任务的 **closure**（闭包）。闭包是那些变量和方法，它们必须是可见的，以便`executor`在`RDD`上执行其计算（在本例中为`foreach()`）。该闭包被序列化并发送给每个`executor`。

闭包的变量副本发给每个 **executor**，当 **counter** 被 `foreach` 函数引用的时候，它已经不再是`driver node`的 **counter** 了。虽然在`driver node`仍然有一个 `counter`在内存中，但是对`executors`已经不可见。`executor`看到的只是序列化的闭包一个副本。所以 **counter** 最终的值还是0，因为对 `counter` 所有的操作均引用序列化的`closure`内的值。

在 `local` 本地模式，在某些情况下的 `foreach` 功能实际上是同一`JVM`上的驱动程序中执行，并会引用同一个原始的 **counter** 计数器，实际上可能更新.

为了确保这些类型的场景明确的行为应该使用的 [`Accumulator`](#accumulators) 累加器。当一个执行的任务分配到集群中的各个`worker`结点时，`Spark` 的累加器是专门提供安全更新变量的机制。本指南的累加器的部分会更详细地讨论这些。

在一般情况下，`closures - constructs`像循环或本地定义的方法，不应该被用于改动一些全局状态。`Spark`没有规定或保证突变的行为，以从封闭件的外侧引用的对象。一些代码，这可能以本地模式运行，但是这只是偶然和这样的代码如预期在分布式模式下不会表现。如果需要一些全局的聚合功能，应使用`Accumulator`（累加器）。

##### 打印RDD的元素

另一种常见的语法用于打印`RDD`的所有元素使用 `rdd.foreach(println)` 或 `rdd.map(println)`。在一台机器上，这将产生预期的输出和打印`RDD`的所有元素。然而，在集群 `cluster` 模式下，`stdout` 输出正在被执行写操作`executors`的 `stdout` 代替，而不是在一个驱动程序上，因此 `stdout` 的 `driver` 程序不会显示这些要打印 `driver` 程序的所有元素，可以使用的 `collect()` 方法首先把`RDD`放到`driver`程序节点上：`rdd.collect().foreach(println)`。这可能会导致`driver`程序耗尽内存，虽说，因为 `collect()` 获取整个`RDD`到一台机器; 如果你只需要打印`RDD`的几个元素，一个更安全的方法是使用 `take()`：`rdd.take(100).foreach(println)`。

### 与键值对一起使用

虽然大多数`Spark`操作都适用于包含任何类型对象的`RDD`，但一些特殊操作仅适用于键值对的`RDD`。 最常见的是分布式"shuffle"操作，例如通过密钥对元素进行分组或聚合。

在`Python`中，这些操作适用于包含内置`Python`元组的`RDD`，例如`（1,2）`。 只需创建这样的元组，然后调用您想要的操作。

例如，以下代码对键值对使用`reduceByKey`操作来计算文件中每行文本出现的次数：

```Python
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)
```

例如，我们也可以使用`counts.sortByKey（）`来按字母顺序对这些对进行排序，最后使用`counts.collect（）`将它们作为对象列表返回给驱动程序。

#### Transformations

下表列出了一些`Spark`常用的`transformations`。详情请参考`RDD API`文档（[`Scala`](api/scala/index.html#org.apache.spark.rdd.RDD)，[`Java`](api/java/index.html?org/apache/spark/api/java/JavaRDD.html)，[`Python`](api/python/pyspark.html#pyspark.RDD)，[`R`](api/R/index.html)）和`pair RDD`函数文档（[`Scala`](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)，[`Java`](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)）。

| Transformation| 含义 |
| --- | --- |
| **map**(_func_) | 返回一个新的`RDD`，它由每个数据源中的元素应用一个函数 _func_ 来生成。 |
| **filter**(_func_) | 返回一个新的`RDD`，它由每个数据源中应用一个函数 _func_ 且返回值为`true`的元素来生成。 |
| **flatMap**(_func_) | 与`map`类似，但是每一个输入的`item`可以被映射成0个或多个输出的`items`（所以 _func_ 应该返回一个`Seq`而不是一个单独的`item`）。 |
| **mapPartitions**(_func_) | 与`map`类似，但是单独的运行在在每个`RDD`的`partition`（分区，`block`）上，所以在一个类型为`T`的`RDD`上运行时 _func_ 必须是 `Iterator<T> => Iterator<U>` 类型。 | 
| **mapPartitionsWithIndex**(_func_) | 与`mapPartitions`类似，但是也需要提供一个代表`partition`的`index`的整型值作为参数的 _func_，所以在一个类型为 `T`的`RDD`上运行时 _func_ 必须是 `(Int, Iterator<T>) =>Iterator<U>`类型。 |
| **sample**(_withReplacement_, _fraction_, _seed_) | 样本数据，设置是否放回（`withReplacement`），采样的百分比（_fraction_）、使用指定的随机数生成器的种子（`seed`）。 | 
| **union**(_otherDataset_) | 反回一个新的`RDD`，它包含了源数据集和其它数据集的并集。 |
| **intersection**(_otherDataset_) | 返回一个新的`RDD`，它包含了源数据集和其它数据集的交集。 |
| **distinct**([_numTasks_])) | 返回一个新的`RDD`，它包含了源数据集中去重的元素。 |
| **groupByKey**([_numTasks_]) | 在一个`(K, V)` 键值对的`RDD`上调用时，返回一个`(K, Iterable<V>)`.**Note:** 如果分组是为了在每一个`key` 上执行聚合操作（例如，`sum`或`average`)，此时使用 `reduceByKey` 或 `aggregateByKey` 来计算性能会更好.**Note:** 默认情况下，并行度取决于父`RDD` 的分区数。可以传递一个可选的 `numTasks` 参数来设置不同的任务数。 |
| **reduceByKey**(_func_, [_numTasks_]) | 在 `(K, V)` 键值对的`RDD`上调用时，返回新的`(K, V)`键值对`RDD`，其中的`values`是针对每个`key`使用给定的函数 _func_ 来进行聚合的，它必须是`type (V,V) => V`的类型。像 `groupByKey` 一样，`reduce tasks`的数量是可以通过第二个可选的参数来配置的。 |
| **aggregateByKey**(_zeroValue_)(_seqOp_, _combOp_, [_numTasks_]) | 在`(K, V)`键值对的 `RDD`上调用时，返回`(K, U)`键值对的`RDD`，其中的`values` 是针对每个`key`使用给定的`combine`函数以及一个`neutral` "0" 值来进行聚合的。允许聚合值的类型与输入值的类型不一样，同时避免不必要的配置。像 `groupByKey` 一样，`reduce tasks`的数量是可以通过第二个可选的参数来配置的。 | 
| **sortByKey**([_ascending_], [_numTasks_]) | 在一个`(K, V)`键值对的`RDD`上调用时，其中的`K`实现了排序，返回一个按`keys`升序或降序的`(K, V)` 键值对的`RDD`，由`boolean`类型的 `ascending` 参数来指定。 |
| **join**(_otherDataset_, [_numTasks_]) | 在一个`(K, V)`和`(K, W)`类型的`RDD`上调用时，返回一个`(K, (V, W))` 键值对的`RDD`，它拥有每个`key` 中所有的元素对。`Outer joins`可以通过 `leftOuterJoin`, `rightOuterJoin` 和 `fullOuterJoin` 来实现。 |
| **cogroup**(_otherDataset_, [_numTasks_]) | 在一个`(K, V)`和`(K, W)`类型的`RDD`上调用时，返回一个`(K, (Iterable<V> Iterable<W>)) tuples`的`RDD`。这个操作也调用了 `groupWith`。 | 
| **cartesian**(_otherDataset_) | 在一个`T`和`U`类型的`RDD`上调用时，返回一个`(T, U)`键值对类型的`RDD`(即笛卡尔积）。 |
| **pipe**(_command_, _[envVars]_) | 通过使用`shell`命令来将每个`RDD`的分区给`Pip`。例如，一个`Perl`或`bash`脚本。`RDD` 的元素会被写入进程的标准输入（`stdin`），并且`lines`（行）输出到它的标准输出（`stdout`）被作为一个字符串型`RDD`的`string`返回。 | 
| **coalesce**(_numPartitions_) | 降低`RDD`中`partitions`（分区）的数量为`numPartition`。对于执行过滤后一个大的`RDD`操作是更有效的。 |
| **repartition**(_numPartitions_) | `Reshuffle`（重新洗牌`RDD`中的数据以创建或者更多的 `partitions`（分区）并将每个分区中的数据尽量保持均匀。该操作总是通过网络来`shuffles`所有的数据。 |
| **repartitionAndSortWithinPartitions**(_partitioner_) | 根据给定的`partitioner`（分区器）对`RDD`进行重新分区，并在每个结果分区中，按照`key` 值对记录排序。这比每一个分区中先调用 `repartition` 然后再排序效率更高，因为它可以将排序过程推送到`shuffle`操作的机器上进行。 |

#### Actions

下表列出了一些`Spark`常用的`actions`操作。详细请参考`RDD API`文档（[`Scala`](api/scala/index.html#org.apache.spark.rdd.RDD)，[`Java`](api/java/index.html?org/apache/spark/api/java/JavaRDD.html)，[`Python`](api/python/pyspark.html#pyspark.RDD)，[`R`](api/R/index.html)）和`RDD`函数文档（[`Scala`](api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)，[`Java`](api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)）。

| Action| 含义|
| --- | --- |
| **reduce**(_func_) | 使用函数 _func_ 聚合`RDD`中的元素，这个函数 _func_ 输入为两个元素，返回为一个元素。这个函数应该是可交换（`commutative`）和关联（`associative`）的，这样才能保证它可以被并行地正确计算。 |
| **collect**() | 在驱动程序中，以一个数组的形式返回`RDD`的所有元素。这在过滤器（`filte`r）或其他操作后返回足够小的数据子集通常是有用的。 |
| **count**() | 返回`RDD`中元素的个数。 |
| **first**() | 返回`RDD`中的第一个元素（类似于`take(1)`)。 |
| **take**(_n_) | 将数据集中的前 _n_ 个元素作为一个数组返回。 |
| **takeSample**(_withReplacement_, _num_, [_seed_]) | 对一个`RDD`进行随机抽样，返回一个包含 _num_ 个随机抽样元素的数组，参数`withReplacement` 指定是否有放回抽样，参数`seed`指定生成随机数的种子。 |
| **takeOrdered**(_n_, _[ordering]_) | 返回`RDD`按自然顺序或自定义比较器排序后的前 _n_ 个元素。 |
| **saveAsTextFile**(_path_) | 将`RDD`中的元素以文本文件（或文本文件集合）的形式写入本地文件系统、`HDFS`或其它`Hadoop`支持的文件系统中的给定目录中。`Spark` 将对每个元素调用`toString`方法，将数据元素转换为文本文件中的一行记录。 |
| **saveAsSequenceFile**(_path_)
(Java and Scala) | 将`RDD`中的元素以`Hadoop SequenceFile`的形式写入到本地文件系统、`HDFS`或其它`Hadoop`支持的文件系统指定的路径中。该操作可以在实现了`Hadoop` 的`Writable`接口的键值对的`RDD`上使用。在`Scala`中，它还可以隐式转换为`Writable`的类型（`Spark`包括了基本类型的转换，例如`Int`，`Double`，`String` 等等)。 |
| **saveAsObjectFile**(_path_)
(Java and Scala) | 使用`Java`序列化以简单的格式编写数据集的元素，然后使用 `SparkContext.objectFile()` 进行加载。 |
| **countByKey**() | 仅适用于`（K,V）`类型的`RDD`。返回具有每个`key`的计数的`（K , Int）`键值对的 `hashmap`。 |
| **foreach**(_func_) | 对`RDD`中每个元素运行函数 _func_。这通常用于副作用，例如更新一个 [Accumulator](#accumulators)（累加器）或与外部存储系统（external storage systems）进行交互。**Note**：修改除 `foreach()` 之外的累加器以外的变量可能会导致未定义的行为。详细介绍请阅读 [Understanding closures（理解闭包）](#understanding-closures-a-nameclosureslinka) 部分。 |

该`Spark RDD API`还暴露了一些`actions`的异步版本，例如针对 `foreach` 的 `foreachAsync`，它们会立即返回一个`FutureAction` 到调用者，而不是在完成`action` 时阻塞。这可以用于管理或等待`action`的异步执行。.

#### Shuffle

`Spark`里的某些操作会触发`shuffle`。`shuffle`是`spark`重新分配数据的一种机制，使得这些数据可以跨不同的区域进行分组。这通常涉及在`executors`和机器之间拷贝数据，这使得`shuffle`成为一个复杂的、代价高的操作。

##### 幕后

为了明白 [`reduceByKey`](#ReduceByLink) 操作的过程，我们以 `reduceByKey` 为例。`reduceBykey`操作产生一个新的`RDD`，其中`key`所有相同的的值组合成为一个`tuple` - `key`以及与`key`相关联的所有值在`reduce`函数上的执行结果。面临的挑战是，一个`key`的所有值不一定都在一个同一个`paritition` 分区里，甚至是不一定在同一台机器里，但是它们必须共同被计算。

在`spark`里，特定的操作需要数据不跨分区分布。在计算期间，一个任务在一个分区上执行，为了所有数据都在单个 `reduceByKey` 的`reduce`任务上运行，我们需要执行一个 `all-to-all`操作。它必须从所有分区读取所有的`key`和`key`对应的所有的值，并且跨分区聚集去计算每个`key`的结果 - 这个过程就叫做 **shuffle**。

尽管每个分区新`shuffle`的数据集将是确定的，分区本身的顺序也是这样，但是这些数据的顺序是不确定的。如果希望`shuffle`后的数据是有序的，可以使用:

*   `mapPartitions` 对每个`partition`分区进行排序，例如，`.sorted`
*   `repartitionAndSortWithinPartitions` 在分区的同时对分区进行高效的排序.
*   `sortBy` 对`RDD`进行全局的排序

触发的`shuffle`操作包括 **repartition** 操作，如 [`repartition`](#RepartitionLink) 和 [`coalesce`](#CoalesceLink)，**‘ByKey** 操作（除了`counting` 之外）像 [`groupByKey`](#GroupByLink) 和 [`reduceByKey`](#ReduceByLink)，和 **join** 操作，像 [`cogroup`](#CogroupLink) 和 [`join`](#JoinLink).

##### 性能影响

该 **Shuffle** 是一个代价比较高的操作，它涉及磁盘`I/O`、数据序列化、网络`I/O`。为了准备`shuffle`操作的数据，`Spark`启动了一系列的任务，_map_ 任务组织数据，_reduce_ 完成数据的聚合。这些术语来自`MapReduce`，跟`Spark`的 `map` 操作和 `reduce` 操作没有关系。

在内部，一个`map`任务的所有结果数据会保存在内存，直到内存不能全部存储为止。然后，这些数据将基于目标分区进行排序并写入一个单独的文件中。在`reduce` 时，任务将读取相关的已排序的数据块。

某些`shuffle`操作会大量消耗堆内存空间，因为`shuffle`操作在数据转换前后，需要在使用内存中的数据结构对数据进行组织。需要特别说明的是，`reduceByKey` 和 `aggregateByKey` 在`map`时会创建这些数据结构，`'ByKey` 操作在`reduce`时创建这些数据结构。当内存满的时候，`Spark`会把溢出的数据存到磁盘上，这将导致额外的磁盘`I/O`开销和垃圾回收开销的增加。

`shuffle`操作还会在磁盘上生成大量的中间文件。在`Spark 1.3`中，这些文件将会保留至对应的`RDD`不在使用并被垃圾回收为止。这么做的好处是，如果在`Spark`重新计算`RDD` 的血统关系时，`shuffle`操作产生的这些中间文件不需要重新创建。如果`Spark`应用长期保持对`RDD` 的引用，或者垃圾回收不频繁，这将导致垃圾回收的周期比较长。这意味着，长期运行`Spark`任务可能会消耗大量的磁盘空间。临时数据存储路径可以通过`SparkContext`中设置参数 `spark.local.dir` 进行配置。

`shuffle`操作的行为可以通过调节多个参数进行设置。详细的说明请看[`Spark`配置指南](configuration.html) 中的 "Shuffle 行为"部分。

### RDD持久化

`Spark`中一个很重要的能力是将数据 _persisting_ 持久化（或称为 _caching_ 缓存），在多个操作间都可以访问这些持久化的数据。当持久化一个`RDD` 时，每个节点的其它分区都可以使用`RDD`在内存中进行计算，在该数据上的其他`action`操作将直接使用内存中的数据。这样会让以后的`action` 操作计算速度加快（通常运行速度会加速10倍）。缓存是迭代算法和快速的交互式使用的重要工具。

`RDD`可以使用 `persist()` 方法或 `cache()` 方法进行持久化。数据将会在第一次`action`操作时进行计算，并缓存在节点的内存中。`Spark`的缓存具有容错机制，如果一个缓存的`RDD`的某个分区丢失了,`Spark`将按照原来的计算过程，自动重新计算并进行缓存。

另外，每个持久化的`RDD`可以使用不同的 _storage level_ 存储级别进行缓存，例如，持久化到磁盘、已序列化的`Java`对象形式持久化到内存（可以节省空间）、跨节点间复制、以 `off-heap`的方式存储在`Tachyon`。这些存储级别通过传递一个 `StorageLevel` 对象（[`Scala`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel)，[`Java`](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/storage/StorageLevel.html)，[`Python`](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.StorageLevel)）给 `persist()` 方法进行设置。`cache()` 方法是使用默认存储级别的快捷设置方法，默认的存储级别是 `StorageLevel.MEMORY_ONLY`（将反序列化的对象存储到内存中）。详细的存储级别介绍如下:

| 存储级别 | 含义 |
| --- | --- |
| MEMORY_ONLY | 将`RDD`以反序列化的`Java`对象的形式存储在`JVM`中。如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算。这是默认的级别。 |
| MEMORY_AND_DISK | 将`RDD`以反序列化的`Java`对象的形式存储在`JVM`中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取。 |
| MEMORY_ONLY_SER
(Java and Scala) | 将`RDD`以序列化的`Java`对象的形式进行存储（每个分区为一个`byte`数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 [fast serializer](https://spark.apache.org/docs/latest/tuning.html) 时会节省更多的空间，但是在读取时会增加CPU的计算负担。 | 
| MEMORY_AND_DISK_SER
(Java and Scala) | 类似于`MEMORY_ONLY_SER`，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算。 |
| DISK_ONLY | 只在磁盘上缓存`RD`。
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc。 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本。 |
| OFF_HEAP（experimental 实验性） | 类似于`MEMORY_ONLY_SER`，但是将数据存储在 [off-heap memory](https://spark.apache.org/docs/latest/configuration.html#memory-management) 中。这需要启用 `off-heap`内存。 |

**Note:** _在 `Python`中，`stored objects will`总是使用 [`Pickle`](https://docs.python.org/2/library/pickle.html) 库来序列化对象，所以无论你选择序列化级别都没关系。在`Python`中可用的存储级别有 `MEMORY_ONLY`，`MEMORY_ONLY_2`，`MEMORY_AND_DISK`，`MEMORY_AND_DISK_2`，`DISK_ONLY`，和 `DISK_ONLY_2`。_

在`shuffle`操作中（例如 `reduceByKey`），即便是用户没有调用 `persist` 方法，`Spark`也会自动缓存部分中间数据.这么做的目的是，在`shuffle` 的过程中某个节点运行失败时，不需要重新计算所有的输入数据。如果用户想多次使用某个`RDD`，强烈推荐在该`RDD`上调用`persist`方法.

#### 如何选择存储级别

`Spark`的存储级别的选择，核心问题是在内存使用率和`CPU`效率之间进行权衡。建议按下面的过程进行存储级别的选择:

*   如果您的`RDD`适合于默认存储级别（`MEMORY_ONLY`），就这样使用。这是`CPU`效率最高的选项，允许`RDD`上的操作尽可能快地运行.

*   如果不是，试着使用 `MEMORY_ONLY_SER` 和 [选择一个快速的序列化库](https://spark.apache.org/docs/latest/tuning.html) 以使对象更加节省空间，但仍然能够快速访问。

*   不要溢出到磁盘，除非计算您的数据集的函数是昂贵的，或者它们过滤大量的数据。否则，重新计算分区可能与从磁盘读取分区一样快.

*   如果需要快速故障恢复，请使用复制的存储级别（例如，如果使用`Spark`来服务 来自网络应用程序的请求）。_All_ 存储级别通过重新计算丢失的数据来提供完整的容错能力，但复制的数据可让您继续在`RDD`上运行任务，而无需等待重新计算一个丢失的分区.

#### 删除数据

`Spark`会自动监视每个节点上的缓存使用情况，并使用`least-recently-used（LRU）`的方式来丢弃旧数据分区。如果您想手动删除`RDD`而不是等待它掉出缓存，使用 `RDD.unpersist()` 方法。

## 共享变量

通常情况下，一个传递给`Spark`操作（例如 `map` 或 `reduce`）的函数是在远程的集群节点上执行的。该函数在多个节点执行过程中使用的变量，是同一个变量的多个副本。这些变量的以副本的方式拷贝到每个机器上，并且各个远程机器上变量的更新并不会传播回驱动程序。通用且支持读-写的共享变量在任务间是不能胜任的。所以，`Spark`提供了两种特定类型的共享变量：`broadcast variables`（广播变量）和` accumulators`（累加器）。

### 广播变量

广播变量允许程序员将一个只读的变量缓存到每台机器上，而不是给任务传递一个副本。它们是如何来使用呢，例如，广播变量可以用一种高效的方式给每个节点传递一份比较大的输入数据集副本。在使用广播变量时，`Spark`也尝试使用高效广播算法分发广播变量以降低通信成本。

`Spark`的 `action`操作是通过一系列的 `stage`进行执行的，这些`stage`是通过分布式的 `shuffle` 操作进行拆分的。`Spark`会自动广播出每个`stage`内任务所需要的公共数据。这种情况下广播的数据使用序列化的形式进行缓存，并在每个任务运行前进行反序列化。这也就意味着，只有在跨越多个`stage`的多个任务会使用相同的数据，或者在使用反序列化形式的数据特别重要的情况下，使用广播变量会有比较好的效果。

广播变量通过在一个变量 `v` 上调用 `SparkContext.broadcast(v)` 方法来进行创建。广播变量是 `v` 的一个包装器，可以通过调用 `value` 方法来访问它的值。代码示例如下:

```Python
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
```

在创建广播变量之后，在集群上执行的所有的函数中，应该使用该广播变量代替原来的 `v` 值，所以节点上的 `v` 最多分发一次。另外，对象 `v` 在广播后不应该再被修改，以保证分发到所有的节点上的广播变量具有同样的值（例如，如果以后该变量会被运到一个新的节点）。

### 累加器

`Accumulators`是一个仅可以通过关联和交换操作执行 `added`的变量来，因此可以高效地执行支持并行。累加器可以用于实现`counter`（计数，类似在`MapReduce` 中那样）或者 `sums`（求和）。原生`Spark`支持数值型的累加器，并且程序员可以添加新的支持类型。

作为一个用户，您可以创建`accumulators`并且重命名。如下图所示，一个命名的 `accumulator`（在这个例子中是 `counter`）将显示在 web UI 中，用于修改该累加器的阶段。`Spark`在 `Tasks`任务表中显示由任务修改的每个累加器的值.

![Accumulators in the Spark UI](img/15dfb146313300f30241c551010cf1a0.jpg "Accumulators in the Spark UI")

在`UI`中跟踪累加器可以有助于了解运行阶段的进度（注：这在 `Python`中尚不支持）.


通过调用`SparkContext.accumulator（v）`从初始值`v`创建累加器。 然后，在集群上运行的任务可以使用`add`方法或`+=`运算符添加到它。 但是，他们无法读懂它的价值。 只有驱动程序才能使用`value`方法读取累加器的值。

下面的代码显示了一个累加器用于添加数组的元素：

```Python
>>> accum = sc.accumulator(0)
>>> accum
Accumulator<id=0, value=0>

>>> sc.parallelize([1, 2, 3, 4]).foreach(lambda x: accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

>>> accum.value
10
```

虽然此代码使用`Int`类型累加器的内置支持，但程序员也可以通过子类化[`AccumulatorParam`]（https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.AccumulatorParam）创建自己的类型。 `AccumulatorParam`接口有两种方法：`zero`用于为数据类型提供"零值"，以及`addInPlace`用于将两个值一起添加。 例如，假设我们有一个表示数学向量的`Vector`类，我们可以写：

```Python
class VectorAccumulatorParam(AccumulatorParam):
    def zero(self, initialValue):
        return Vector.zeros(initialValue.size)

    def addInPlace(self, v1, v2):
        v1 += v2
        return v1

# Then, create an Accumulator of this type:
vecAccum = sc.accumulator(Vector(...), VectorAccumulatorParam())
```

累加器的更新只发生在 **action** 操作中，`Spark`保证每个任务只更新累加器一次，例如，重启任务不会更新值。在 `transformation`中，用户需要注意的是，如果 `task`或 `job stages`重新执行，每个任务的更新操作可能会执行多次。

累加器不会改变 `Spark`懒加载的模式。如果累加器在`RDD`中的一个操作中进行更新，它们的值仅被更新一次，`RDD`被作为`action` 的一部分来计算。因此，在一个像 `map()` 这样的 `transformation`时，累加器的更新并没有执行。下面的代码片段证明了这个特性:

```Python
accum = sc.accumulator(0)
def g(x):
    accum.add(x)
    return f(x)
data.map(g)
# Here, accum is still 0 because no actions have caused the `map` to be computed.
```

## 部署应用到集群中

该[应用提交指南](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/launcher/package-summary.html) 描述了如何将应用提交到集群中。简单的说，在您将应用打包成一个`JAR`（针对`Java/Scala`）或者一组 `.py` 或 `.zip` 文件（针对 `Python`），该 `bin/spark-submit` 脚本可以让你提交它到任何所支持的 `cluster manager`上去.

## 从Java/Scala启动Spark-jobs

该[`org.apache.spark.launcher`](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/launcher/package-summary.html) `package `提供了`classes`用于使用简单的`Java API`来作为一个子进程启动`Spark jobs`.

## 单元测试

`Spark`可以友好的使用流行的单元测试框架进行单元测试。在将`master URL`设置为 `local` 来测试时会简单的创建一个 `SparkContext`，运行您的操作，然后调用 `SparkContext.stop()` 将该作业停止。因为`Spark`不支持在同一个程序中并行的运行两个`contexts`，所以需要确保使用 `finally`块或者测试框架的 `tearDown` 方法将 `context`停止。

## 快速链接

您可以在`Spark`网站上看一下 [`Spark`程序示例](http://spark.apache.org/examples.html)。此外`Spark`在 `examples` 目录中包含了许多示例（[`Scala`](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples)，[`Java`](https://github.com/apache/spark/tree/master/examples/src/main/java/org/apache/spark/examples)，[`Python`](https://github.com/apache/spark/tree/master/examples/src/main/python)，[`R`](https://github.com/apache/spark/tree/master/examples/src/main/r)）。您可以通过传递`class name`到`Spark`的`bin/run-example`脚本以运行`Java`和`Scala`示例; 例如:

```bash
./bin/run-example SparkPi 
```

针对`Python`示例，使用 `spark-submit` 来代替:

```bash
./bin/spark-submit examples/src/main/python/pi.py 
```

针对`R`示例，使用 `spark-submit` 来代替:

```bash
./bin/spark-submit examples/src/main/r/dataframe.R 
```

针对应用程序的优化，该[配置](configuration.html)和[优化](tuning.html) 指南一些最佳实践的信息。这些优化建议在确保你的数据以高效的格式存储在内存中尤其重要。针对部署参考，该[集群模式概述](cluster-overview.html) 描述了分布式操作和支持的集群管理器的组件.

最后,所有的`API`文档可在[`Scala`](api/scala/#org.apache.spark.package)，[`Java`](api/java/)，[`Python`](api/python/)和[`R`](api/R/) 中获取.