# 快速入门
*   [安全](#安全)
*   [使用 Spark Shell 进行交互式分析](#使用-spark-shell-进行交互式分析)
    *   [基础](#基础)
    *   [Dataset 上的更多操作](#dataset-上的更多操作)
    *   [缓存](#缓存)
*   [独立的应用](#独立的应用)
*   [快速跳转](#快速跳转)

本教程提供了如何使用`Spark`的快速入门介绍。首先通过运行`Spark`交互式的`shell`（在`Python`或`Scala`中）来介绍`API`，然后展示如何使用`Java`，`Scala` 和`Python` 来编写应用程序。

为了继续阅读本指南，首先从[Spark 官网](http://spark.apache.org/downloads.html) 下载`Spark`的发行包。因为我们将不使用`HDFS`，所以你可以下载一个任何`Hadoop` 版本的软件包。

请注意，在`Spark 2.0`之前，`Spark`的主要编程接口是弹性分布式数据集（`RDD`）。 在`Spark 2.0`之后，`RDD`被`Dataset`替换，它是像`RDD`一样的 `strongly-typed`（强类型），但是在引擎盖下更加优化。 `RDD`接口仍然受支持，您可以在[`RDD`编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html) 中获得更完整的参考。 但是，我们强烈建议您切换到使用`Dataset`（数据集），其性能要更优于`RDD`。请参阅[SQL 编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html) 获取更多有关`Dataset`的信息。

# 安全
`Spark`中的安全性默认为`OF`F。 这可能意味着您很容易受到默认攻击。 在下载和运行`Spark`之前，请参阅[Spark Security](https://spark.apache.org/docs/latest/security.html)。

# 使用Spark Shell进行交互式分析

## 基础

`Spark shell`提供了一种来学习该`API`比较简单的方式，以及一个强大的来分析数据交互的工具。在`Scala`（运行于`Java`虚拟机之上，并能很好的调用已存在的`Java`类库）或者 `Python`中它是可用的。通过在`Spark`目录中运行以下的命令来启动它:

```
./bin/pyspark
```
或者如果在当前环境中使用`pip`安装了`PySpark`：
```
pyspark
```

`Spark`的主要抽象是一个称为`Dataset`的分布式的`item`集合。`Datasets`可以从`Hadoop`的`InputFormats`（例如 HDFS文件）或者通过其它的`Datasets`转换来创建。由于`Python`的动态特性，我们不需要在`Pytho`n中强类型数据集。 因此，`Python`中的所有数据集都是`Dataset [Row]`，我们称之为`DataFrame`与`Pandas`和`R`中的数据框概念一致。让我们从`Spark`源目录中的`README`文件来创建一个新的`Dataset`:

```
>>> textFile = spark.read.text("README.md")
```

您可以直接从`Dataset`中获取`values`（值），通过调用一些`actions`（动作），或者`transform`（转换）`Dataset`以获得一个新的。更多细节，请参阅 _[API doc](https://spark.apache.org/docs/latest/api/python/index.html#pyspark.sql.DataFrame)_。

```
>>> textFile.count()  # Number of rows in this DataFrame
126

>>> textFile.first()  # First row in this DataFrame
Row(value=u'# Apache Spark')
```

现在让我们`transform`这个`Dataset`以获得一个新的。我们调用 `filter` 以返回一个新的`Dataset`，它是文件中的`items`的一个子集。

```
>>> linesWithSpark = textFile.filter(textFile.value.contains("Spark"))
```

我们可以链式操作`transformation`（转换）和`action`（动作）:

```
>>> textFile.filter(textFile.value.contains("Spark")).count()  # How many lines contain "Spark"?
15
```

## Dataset上的更多操作

`Dataset actions`（操作）和`transformations`（转换）可以用于更复杂的计算。例如，统计出现次数最多的行 :

```
>>> from pyspark.sql.functions import *
>>> textFile.select(size(split(textFile.value, "\s+")).name("numWords")).agg(max(col("numWords"))).collect()
[Row(max(numWords)=15)]
```

首先将一行映射为整数值，并将其别名为"numWords"，从而创建一个新的`DataFrame`。 在该`DataFrame`上调用`agg`以查找最大字数。 `select`和`agg`的参数都是`Column`，我们可以使用`df.colName`从`DataFrame`中获取一列。 我们还可以导入`pyspark.sql.functions`，它提供了许多方便的功能来从旧的列构建一个新的列。

一种常见的数据流模式是被`Hadoop`所推广的`MapReduce`。`Spark`可以很容易实现`MapReduce`:

```
>>> wordCounts = textFile.select(explode(split(textFile.value, "\s+")).alias("word")).groupBy("word").count()
```

在这里，我们使用`select`中的`explode`函数，将行数据集转换为单词数据集，然后将`groupBy`和`count`结合起来计算文件中的每个单词计数，作为2列的`DataFrame`："word"和"count"。 要在我们的`shell`中收集单词`count`，我们可以调用`collect`：

```
>>> wordCounts.collect()
[Row(word=u'online', count=1), Row(word=u'graphs', count=1), ...]
```

## 缓存

`Spark`还支持`Pulling`（拉取）数据集到一个群集范围的内存缓存中。这在反复访问数据时非常有用，例如当查询一个小的 "hot" 数据集或运行一个像`PageRANK`这样的迭代算法时，在数据被重复访问时是非常高效的。举一个简单的例子，让我们标记我们的 `linesWithSpark` 数据集到缓存中:

```
>>> linesWithSpark.cache()

>>> linesWithSpark.count()
15

>>> linesWithSpark.count()
15
```

使用`Spark`来探索和缓存一个100行的文本文件看起来比较愚蠢。有趣的是，即使在他们跨越几十或者几百个节点时，这些相同的函数也可以用于非常大的数据集。您也可以像 [编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html#using-the-shell). 中描述的一样通过连接 `bin/spark-shell` 到集群中，使用交互式的方式来做这件事情。

# 独立的应用

假设我们希望使用`Spark API`来创建一个独立的应用程序。我们在`Scala(SBT)`，`Java(Maven)`和`Python(pip)`中练习一个简单应用程序。

现在我们将展示如何使用`Python API(PySpark)`编写应用程序。

如果要构建打包的`PySpark`应用程序或库，可以将其添加到`setup.py`文件中：
```
    install_requires=[
        'pyspark=={site.SPARK_VERSION}'
    ]
```

作为示例，我们将创建一个简单的`Spark`应用程序, `SimpleApp.py`:

```
"""SimpleApp.py"""
from pyspark.sql import SparkSession

logFile = "YOUR_SPARK_HOME/README.md"  # Should be some file on your system
spark = SparkSession.builder().appName(appName).master(master).getOrCreate()
logData = spark.read.text(logFile).cache()

numAs = logData.filter(logData.value.contains('a')).count()
numBs = logData.filter(logData.value.contains('b')).count()

print("Lines with a: %i, lines with b: %i" % (numAs, numBs))

spark.stop()
```

该程序只计算包含'a'的行数和包含文本文件中'b'的数字。 请注意，您需要将`YOUR_SPARK_HOME`替换为安装`Spark`的位置。 与`Scala`和`Java`示例一样，我们使用`SparkSession`来创建数据集。 对于使用自定义类或第三方库的应用程序，我们还可以通过将它们打包到`.zi`p文件中来添加代码依赖关系以通过其`--py-files`参数进行`spark-submit`（有关详细信息，请参阅`spark-submit --help`）。 `SimpleApp`非常简单，我们不需要指定任何代码依赖项。

我们可以使用`bin/spark-submit`脚本运行此应用程序：

```
# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --master local[4] \
  SimpleApp.py
...
Lines with a: 46, Lines with b: 23
```
如果您的环境中安装了`PySpark pip`（例如，`pip install pyspar`k），您可以使用常规`Pytho`n解释器运行您的应用程序，或者根据您的喜好使用提供的`spark-submit`。
```
# Use the Python interpreter to run your application
$ python SimpleApp.py
...
Lines with a: 46, Lines with b: 23
```

# 快速跳转

恭喜您成功的运行了您的第一个`Spark`应用程序！

*   更多`API`的深入概述，从[RDD programming guide](https://spark.apache.org/docs/latest/rdd-programming-guide.html) 和 [SQL programming guide](https://spark.apache.org/docs/latest/sql-programming-guide.html) 这里开始，或者看看 “编程指南” 菜单中的其它组件。
*   为了在集群上运行应用程序，请前往 [deployment overview](https://spark.apache.org/docs/latest/cluster-overview.html).
*   最后，在`Spark`的 `examples` 目录中包含了一些 ([Scala](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples)，[Java](https://github.com/apache/spark/tree/master/examples/src/main/java/org/apache/spark/examples)，[Python](https://github.com/apache/spark/tree/master/examples/src/main/python)，[R](https://github.com/apache/spark/tree/master/examples/src/main/r)) 示例。您可以按照如下方式来运行它们:

```
# 针对 Scala 和 Java，使用 run-example:
./bin/run-example SparkPi

# 针对 Python 示例，直接使用 spark-submit:
./bin/spark-submit examples/src/main/python/pi.py

# 针对 R 示例，直接使用 spark-submit:

./bin/spark-submit examples/src/main/r/dataframe.R
```