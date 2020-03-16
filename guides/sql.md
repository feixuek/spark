# Spark SQL, DataFrames and Datasets Guide

*   [总览](#总览)
    *   [SQL](#SQL)
    *   [Datasets和DataFrames](#Datasets和DataFrames)
*   [开始入门](#开始入门)
    *   [起始点:SparkSession](#起始点:SparkSession)
    *   [创建DataFrames](#创建DataFrames)
    *   [无类型的Dataset操作](#无类型的Dataset操作)
    *   [以编程方式运行SQL查询](#以编程方式运行SQL查询)
    *   [全局临时视图](#全局临时视图)
    *   [创建Datasets](#创建Datasets)
    *   [RDD的互操作性](#RDD的互操作性)
        *   [使用反射推断Schema](#使用反射推断Schema)
        *   [以编程的方式指定Schema](#以编程的方式指定Schema)
    *   [聚合](#聚合)
        *   [无类型的用户定义聚合函数](#无类型的用户定义聚合函数)
        *   [类型安全的用户定义聚合函数](#类型安全的用户定义聚合函数)
*   [数据源](#数据源)
    *   [通用加载/保存方法](#通用加载/保存方法)
        *   [手动指定选项](#手动指定选项)
        *   [直接在文件上运行SQL](#直接在文件上运行SQL)
        *   [保存模式](#保存模式)
        *   [保存到持久化表](#保存到持久化表)
        *   [分桶，排序和分区](#分桶，排序和分区])
    *   [Parquet文件](#Parquet文件)
        *   [以编程的方式加载数据](#以编程的方式加载数据)
        *   [分区发现](#分区发现)
        *   [模式合并](#模式合并)
        *   [Hive-metastore-Parquet表转换](#Hive-metastore-Parquet表转换)
            *   [Hive/Parquet-Schema协调](#Hive/Parquet-Schema协调)
            *   [元数据刷新](#元数据刷新)
        *   [配置](#配置)
    *   [ORC文件](#ORC文件)
    *   [JSON数据集](#JSON数据集)
    *   [Hive表](#Hive表)
        *   [指定Hive表的存储格式](#指定Hive表的存储格式)
        *   [与不同版本的Hive-Metastore进行交互](#与不同版本的Hive-Metastore进行交互)
    *   [JDBC连接其它数据库](#JDBC连接其它数据库)
    *   [故障排除](#故障排除)
*   [性能调优](#性能调优)
    *   [在内存中缓存数据](#在内存中缓存数据)
    *   [其他配置选项](#其他配置选项)
    *   [SQL查询的广播提示](#SQL查询的广播提示)
*   [分布式SQL引擎](#分布式SQL引擎)
    *   [运行Thrift-JDBC/ODBC服务器](#运行Thrift-JDBC/ODBC服务器)
    *   [运行Spark-SQL-CLI](#运行Spark-SQL-CLI)
*   [使用Apache-Arrow的Pandas-PySpark使用指南](#使用Apache-Arrow的Pandas-PySpark使用指南)
    *   [Spark中的Apache-Arrow](#Spark中的Apache-Arrow)
        *   [确保PyArrow已安装](#确保PyArrow已安装)
    *   [启用与Pandas的转换](启用与Pandas的转换)
    *   [Pandas-UDF）](#Pandas-UDF)
        *   [标量](#标量)
        *   [分组映射](#分组映射)
        *   [分组聚合](#分组聚合)
    *   [使用笔记](#使用笔记)
        *   [支持的SQL类型](#支持的SQL类型)
        *   [设置箭头批量大小](#设置箭头批量大小)
        *   [带时区语义的时间戳](#带时区语义的时间戳)
*   [迁移指南](#迁移指南)
*   [参考](#参考)
    *   [数据类型](#数据类型)
    *   [NaN语义学](#NaN语义学)
    *   [算术运算](#算术运算)

# 总览

`Spark SQL`是`Spark`处理结构化数据的一个模块。与基础的`Spark RDD API`不同，`Spark SQL`提供了查询结构化数据及计算结果等信息的接口。在内部，`Spark SQL` 使用这个额外的信息去执行额外的优化。有几种方式可以跟`Spark SQL`进行交互，包括`SQL`和`Dataset API`。当使用相同执行引擎进行计算时，无论使用哪种`API/语言`都可以快速的计算。这种统一意味着开发人员可以轻松地在不同的`API`之间来回切换，从而提供表达给定转换的最自然的方式。

该页面所有例子使用的示例数据都包含在`Spark`的发布中，并且可以使用 `spark-shell`，`pyspark shell`，或者 `sparkR shell`来运行。

## SQL

`Spark SQL`的功能之一是执行SQL查询。`Spark SQL`也能够被用于从已存在的`Hive`环境中读取数据。更多关于如何配置这个特性的信息，请参考[`Hive`表](https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html) 这部分。当以另外的编程语言运行`SQL`时，查询结果将以[`Dataset/DataFrame`](#Datasets和DataFrames)的形式返回。您也可以使用[命令行](https://spark.apache.org/docs/latest/sql-distributed-sql-engine.html)或者通过[`JDBC/ODBC`](https://spark.apache.org/docs/latest/sql-distributed-sql-engine.html)与`SQL`接口交互。

## Datasets和DataFrames

一个`Dataset`是一个分布式的数据集合。`Dataset`是在`Spark 1.6`中被添加的新接口，它提供了`RDD`的优点（强类型化，能够使用强大的`lambda`函数）与`Spark SQL`执行引擎的优点。一个`Dataset`可以从`JVM`对象来[构造](https://spark.apache.org/docs/latest/sql-getting-started.html) 并且使用转换功能（`map`，`flatMap`，`filter`，等等）。`Dataset API`在[`Scala`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset) 和 [`Java`](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html)中是可用的。`Python`不支持`Dataset API`。但是由于`Python`的动态特性，许多`Dataset API`的优点已经可用了（也就是说，您可以通过名称自然地访问行的字段`row.columnName`）。这种情况和`R`相似。

一个`DataFrame`是一个 _Dataset_ 组成的指定列。它的概念与一个在关系型数据库或者在`R/Python`中的表是相等的，但是有很多优化。`DataFrames`可以从大量的 [源](https://spark.apache.org/docs/latest/sql-data-sources.html) 中构造出来，比如：结构化的文本文件，`Hive`中的表，外部数据库，或者已经存在的`RDDs`。`DataFrame API`可以在`Scala`，`Java`，[`Python`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame)，和 [`R`](https://spark.apache.org/docs/latest/api/R/index.html)中实现。在 `Scala` 和 `Java`中，`DataFrame`由`DataSet`中的 `RowS`（多个`Row`）来表示。在 [`Scala API`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)中，`DataFrame` 仅仅是一个 `Dataset[Row]`类型的别名。然而，在 [`Java API`](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html)中，用户需要去使用 `Dataset<Row>` 去代表一个 `DataFrame`。

在此文档中，我们将常常会引用`Scala/Java Datasets`的 `Row`s 作为`DataFrames`。

# 开始入门

## 起始点:SparkSession

`Spark SQL`中所有功能的入口点是 [`SparkSession`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.SparkSession) 类。要创建一个 `SparkSession`，仅使用 `SparkSession.builder`就可以了:

```Python
from pyspark.sql import SparkSession

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

`Spark 2.0`中的`SparkSession` 为`Hive`特性提供了内嵌的支持，包括使用`HiveQL`编写查询的能力，访问`Hive UDF`,以及从`Hive` 表中读取数据的能力。为了使用这些特性，你不需要去有一个已存在的`Hive`设置。

## 创建DataFrames

在一个 `SparkSession`中，应用程序可以从一个 [已经存在的 `RDD`](https://spark.apache.org/docs/latest/sql-getting-started.html#interoperating-with-rdds)，从`hive`表，或者从 [`Spark`数据源](https://spark.apache.org/docs/latest/sql-data-sources.html)中创建一个`DataFrames`。

举个例子，下面就是基于一个JSON文件创建一个DataFrame:

```Python
# spark is an existing SparkSession
df = spark.read.json("examples/src/main/resources/people.json")
# Displays the content of the DataFrame to stdout
df.show()
# +----+-------+
# | age|   name|
# +----+-------+
# |null|Michael|
# |  30|   Andy|
# |  19| Justin|
# +----+-------+

```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

## 无类型的Dataset操作

`DataFrames`提供了一个特定的语法用在 [`Scala`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset)，[`Java`](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html)，[`Python`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame) 和[`R`](https://spark.apache.org/docs/latest/api/R/SparkDataFrame.html)中机构化数据的操作。

正如上面提到的一样，`Spark 2.0` 中，`DataFrames`在`Scala`和`Java API`中，仅仅是多个 `Row`s 的`Dataset`。这些操作也参考了与强类型的`Scala/Java Datasets`中的 "类型转换"" 对应的 "无类型转换"。

这里包括一些使用 `Dataset` 进行结构化数据处理的示例 :

在`Python`中，可以通过(`df.age`) 或者(`df['age']`)来获取`DataFrame`的列。虽然前者便于交互式操作，但是还是建议用户使用后者，这样不会破坏列名，也能引用`DataFrame`的类。

```Python
# spark, df are from the previous example
# Print the schema in a tree format
df.printSchema()
# root
# |-- age: long (nullable = true)
# |-- name: string (nullable = true)

# Select only the "name" column
df.select("name").show()
# +-------+
# |   name|
# +-------+
# |Michael|
# |   Andy|
# | Justin|
# +-------+

# Select everybody, but increment the age by 1
df.select(df['name'], df['age'] + 1).show()
# +-------+---------+
# |   name|(age + 1)|
# +-------+---------+
# |Michael|     null|
# |   Andy|       31|
# | Justin|       20|
# +-------+---------+

# Select people older than 21
df.filter(df['age'] > 21).show()
# +---+----+
# |age|name|
# +---+----+
# | 30|Andy|
# +---+----+

# Count people by age
df.groupBy("age").count().show()
# +----+-----+
# | age|count|
# +----+-----+
# |  19|    1|
# |null|    1|
# |  30|    1|
# +----+-----+
```

<small>可以在`Spark`仓库找到完整例子 "examples/src/main/python/sql/basic.py"</small>

为了能够在`DataFrame`上被执行的操作类型的完整列表请参考[`API`文档](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame)。

除了简单的列引用和表达式之外，`DataFrame`也有丰富的函数库，包括`string`操作，`date`算术，常见的`math`操作以及更多。可用的完整列表请参考[`DataFrame`函数指南](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#module-pyspark.sql.functions)。

## 以编程方式运行SQL查询

`SparkSession` 的 `sql` 函数可以让应用程序以编程的方式运行`SQL`查询，并将结果作为一个 `DataFrame` 返回。

```Python
# Register the DataFrame as a SQL temporary view
df.createOrReplaceTempView("people")

sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
# +----+-------+
# | age|   name|
# +----+-------+
# |null|Michael|
# |  30|   Andy|
# |  19| Justin|
# +----+-------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

## 全局临时视图

`Spark SQL`中的临时视图是`session`级别的，也就是会随着`session`的消失而消失。如果你想让一个临时视图在所有`session`中相互传递并且可用，直到`Spark`应用退出，你可以建立一个全局的临时视图。全局的临时视图存在于系统数据库 `global_temp`中，我们必须加上库名去引用它，比如。 `SELECT * FROM global_temp.view1`。


```Python
# Register the DataFrame as a global temporary view
df.createGlobalTempView("people")

# Global temporary view is tied to a system preserved database `global_temp`
spark.sql("SELECT * FROM global_temp.people").show()
# +----+-------+
# | age|   name|
# +----+-------+
# |null|Michael|
# |  30|   Andy|
# |  19| Justin|
# +----+-------+

# Global temporary view is cross-session
spark.newSession().sql("SELECT * FROM global_temp.people").show()
# +----+-------+
# | age|   name|
# +----+-------+
# |null|Michael|
# |  30|   Andy|
# |  19| Justin|
# +----+-------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

## 创建Datasets

`Dataset`与`RDD`相似，然而，并不是使用`Java` 序列化或者`Kryo`[编码器](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Encoder) 来序列化用于处理或者通过网络进行传输的对象。虽然编码器和标准的序列化都负责将一个对象序列化成字节，编码器是动态生成的代码，并且使用了一种允许`Spark`去执行许多像 `filtering`，`sorting` 以及 `hashing` 这样的操作，不需要将字节反序列化成对象的格式。

```Scala
// Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
// you can use custom classes that implement the Product interface
case class Person(name: String, age: Long)

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

<small>可以在`Spark`仓库找到完整例子 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala"</small>

```Java
import java.util.Arrays;
import java.util.Collections;
import java.io.Serializable;

import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

public static class Person implements Serializable {
  private String name;
  private int age;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }
}

// Create an instance of a Bean class
Person person = new Person();
person.setName("Andy");
person.setAge(32);

// Encoders are created for Java beans
Encoder<Person> personEncoder = Encoders.bean(Person.class);
Dataset<Person> javaBeanDS = spark.createDataset(
  Collections.singletonList(person),
  personEncoder
);
javaBeanDS.show();
// +---+----+
// |age|name|
// +---+----+
// | 32|Andy|
// +---+----+

// Encoders for most common types are provided in class Encoders
Encoder<Integer> integerEncoder = Encoders.INT();
Dataset<Integer> primitiveDS = spark.createDataset(Arrays.asList(1, 2, 3), integerEncoder);
Dataset<Integer> transformedDS = primitiveDS.map(
    (MapFunction<Integer, Integer>) value -> value + 1,
    integerEncoder);
transformedDS.collect(); // Returns [2, 3, 4]

// DataFrames can be converted to a Dataset by providing a class. Mapping based on name
String path = "examples/src/main/resources/people.json";
Dataset<Person> peopleDS = spark.read().json(path).as(personEncoder);
peopleDS.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java"</small>

## RDD的互操作性

`Spark SQL`支持两种不同的方法用于转换已存在的`RDD`成为`Dataset`。第一种方法是使用反射去推断一个包含指定的对象类型的`RDD`的`Schema`。在你的`Spark` 应用程序中当你已知`Schema`时这个基于方法的反射可以让你的代码更简洁。

第二种用于创建`Dataset`的方法是通过一个允许你构造一个`Schema`然后把它应用到一个已存在的`RDD` 的编程接口。然而这种方法更繁琐，当列和它们的类型知道运行时都是未知时它允许你去构造`Dataset`。

### 使用反射推断Schema

`Spark SQL`能够把`RDD`转换为一个`DataFrame`，并推断其类型。这些行由一系列`key/value`键值对组成。`key`值代表了表的列名,类型按抽样推断整个数据集，同样的也适用于`JSON`文件。

```Python
from pyspark.sql import Row

sc = spark.sparkContext

# Load a text file and convert each line to a Row.
lines = sc.textFile("examples/src/main/resources/people.txt")
parts = lines.map(lambda l: l.split(","))
people = parts.map(lambda p: Row(name=p[0], age=int(p[1])))

# Infer the schema, and register the DataFrame as a table.
schemaPeople = spark.createDataFrame(people)
schemaPeople.createOrReplaceTempView("people")

# SQL can be run over DataFrames that have been registered as a table.
teenagers = spark.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")

# The results of SQL queries are Dataframe objects.
# rdd returns the content as an :class:`pyspark.RDD` of :class:`Row`.
teenNames = teenagers.rdd.map(lambda p: "Name: " + p.name).collect()
for name in teenNames:
    print(name)
# Name: Justin
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

### 以编程的方式指定Schema

当一个字典不能被提前定义（例如，记录的结构是在一个字符串中，抑或一个文本中解析，被不同的用户所属），一个 `DataFrame` 可以通过以下3步来创建。

1.  `RDD`从原始的`RDD`创建一个`RDD`的 `tuples`或者一个`lists`;
2.  `Step 1` 被创建后，创建`Schema`表示一个 `StructType` 匹配`RDD`中的结构。
3.  通过 `SparkSession` 提供的 `createDataFrame` 方法应用`Schema`到`RDD`。

例子:

```Python
# Import data types
from pyspark.sql.types import *

sc = spark.sparkContext

# Load a text file and convert each line to a Row.
lines = sc.textFile("examples/src/main/resources/people.txt")
parts = lines.map(lambda l: l.split(","))
# Each line is converted to a tuple.
people = parts.map(lambda p: (p[0], p[1].strip()))

# The schema is encoded in a string.
schemaString = "name age"

fields = [StructField(field_name, StringType(), True) for field_name in schemaString.split()]
schema = StructType(fields)

# Apply the schema to the RDD.
schemaPeople = spark.createDataFrame(people, schema)

# Creates a temporary view using the DataFrame
schemaPeople.createOrReplaceTempView("people")

# SQL can be run over DataFrames that have been registered as a table.
results = spark.sql("SELECT name FROM people")

results.show()
# +-------+
# |   name|
# +-------+
# |Michael|
# |   Andy|
# | Justin|
# +-------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/basic.py"</small>

## 聚合

[内建`DataFrames`函数](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction)提供常见的聚合，如 `count()`, `countDistinct()`, `avg()`, `max()`, `min()`, etc. 虽然这些函数是为`DataFrames`设计的，但`Spark SQL`也有一些类型安全的版本[`Scala`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.scalalang.typed$)和[`Java`](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/expressions/javalang/typed.html) 使用强类型数据集。 此外，用户不限于预定义的聚合函数，并且可以创建自己的聚合函数。

### 无类型的用户定义聚合函数

用户必须扩展[UserDefinedAggregateFunction](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.UserDefinedAggregateFunction)抽象类以实现自定义无类型聚合函数。 例如，用户定义的平均值可能如下所示：

```Scala
import org.apache.spark.sql.expressions.MutableAggregationBuffer
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row
import org.apache.spark.sql.SparkSession

object MyAverage extends UserDefinedAggregateFunction {
  // Data types of input arguments of this aggregate function
  def inputSchema: StructType = StructType(StructField("inputColumn", LongType) :: Nil)
  // Data types of values in the aggregation buffer
  def bufferSchema: StructType = {
    StructType(StructField("sum", LongType) :: StructField("count", LongType) :: Nil)
  }
  // The data type of the returned value
  def dataType: DataType = DoubleType
  // Whether this function always returns the same output on the identical input
  def deterministic: Boolean = true
  // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
  // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
  // the opportunity to update its values. Note that arrays and maps inside the buffer are still
  // immutable.
  def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = 0L
    buffer(1) = 0L
  }
  // Updates the given aggregation buffer `buffer` with new input data from `input`
  def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    if (!input.isNullAt(0)) {
      buffer(0) = buffer.getLong(0) + input.getLong(0)
      buffer(1) = buffer.getLong(1) + 1
    }
  }
  // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
  def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getLong(0) + buffer2.getLong(0)
    buffer1(1) = buffer1.getLong(1) + buffer2.getLong(1)
  }
  // Calculates the final result
  def evaluate(buffer: Row): Double = buffer.getLong(0).toDouble / buffer.getLong(1)
}

// Register the function to access it
spark.udf.register("myAverage", MyAverage)

val df = spark.read.json("examples/src/main/resources/employees.json")
df.createOrReplaceTempView("employees")
df.show()
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

val result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees")
result.show()
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedUntypedAggregation.scala" </small>

```Java
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.expressions.MutableAggregationBuffer;
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction;
import org.apache.spark.sql.types.DataType;
import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

public static class MyAverage extends UserDefinedAggregateFunction {

  private StructType inputSchema;
  private StructType bufferSchema;

  public MyAverage() {
    List<StructField> inputFields = new ArrayList<>();
    inputFields.add(DataTypes.createStructField("inputColumn", DataTypes.LongType, true));
    inputSchema = DataTypes.createStructType(inputFields);

    List<StructField> bufferFields = new ArrayList<>();
    bufferFields.add(DataTypes.createStructField("sum", DataTypes.LongType, true));
    bufferFields.add(DataTypes.createStructField("count", DataTypes.LongType, true));
    bufferSchema = DataTypes.createStructType(bufferFields);
  }
  // Data types of input arguments of this aggregate function
  public StructType inputSchema() {
    return inputSchema;
  }
  // Data types of values in the aggregation buffer
  public StructType bufferSchema() {
    return bufferSchema;
  }
  // The data type of the returned value
  public DataType dataType() {
    return DataTypes.DoubleType;
  }
  // Whether this function always returns the same output on the identical input
  public boolean deterministic() {
    return true;
  }
  // Initializes the given aggregation buffer. The buffer itself is a `Row` that in addition to
  // standard methods like retrieving a value at an index (e.g., get(), getBoolean()), provides
  // the opportunity to update its values. Note that arrays and maps inside the buffer are still
  // immutable.
  public void initialize(MutableAggregationBuffer buffer) {
    buffer.update(0, 0L);
    buffer.update(1, 0L);
  }
  // Updates the given aggregation buffer `buffer` with new input data from `input`
  public void update(MutableAggregationBuffer buffer, Row input) {
    if (!input.isNullAt(0)) {
      long updatedSum = buffer.getLong(0) + input.getLong(0);
      long updatedCount = buffer.getLong(1) + 1;
      buffer.update(0, updatedSum);
      buffer.update(1, updatedCount);
    }
  }
  // Merges two aggregation buffers and stores the updated buffer values back to `buffer1`
  public void merge(MutableAggregationBuffer buffer1, Row buffer2) {
    long mergedSum = buffer1.getLong(0) + buffer2.getLong(0);
    long mergedCount = buffer1.getLong(1) + buffer2.getLong(1);
    buffer1.update(0, mergedSum);
    buffer1.update(1, mergedCount);
  }
  // Calculates the final result
  public Double evaluate(Row buffer) {
    return ((double) buffer.getLong(0)) / buffer.getLong(1);
  }
}

// Register the function to access it
spark.udf().register("myAverage", new MyAverage());

Dataset<Row> df = spark.read().json("examples/src/main/resources/employees.json");
df.createOrReplaceTempView("employees");
df.show();
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

Dataset<Row> result = spark.sql("SELECT myAverage(salary) as average_salary FROM employees");
result.show();
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedUntypedAggregation.java"</small>

### 类型安全的用户定义聚合函数

强类型数据集的用户定义聚合围绕[`Aggregator`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.expressions.Aggregator)抽象类。 例如，类型安全的用户定义平均值可能如下所示：

```Scala
import org.apache.spark.sql.expressions.Aggregator
import org.apache.spark.sql.Encoder
import org.apache.spark.sql.Encoders
import org.apache.spark.sql.SparkSession

case class Employee(name: String, salary: Long)
case class Average(var sum: Long, var count: Long)

object MyAverage extends Aggregator[Employee, Average, Double] {
  // A zero value for this aggregation. Should satisfy the property that any b + zero = b
  def zero: Average = Average(0L, 0L)
  // Combine two values to produce a new value. For performance, the function may modify `buffer`
  // and return it instead of constructing a new object
  def reduce(buffer: Average, employee: Employee): Average = {
    buffer.sum += employee.salary
    buffer.count += 1
    buffer
  }
  // Merge two intermediate values
  def merge(b1: Average, b2: Average): Average = {
    b1.sum += b2.sum
    b1.count += b2.count
    b1
  }
  // Transform the output of the reduction
  def finish(reduction: Average): Double = reduction.sum.toDouble / reduction.count
  // Specifies the Encoder for the intermediate value type
  def bufferEncoder: Encoder[Average] = Encoders.product
  // Specifies the Encoder for the final output value type
  def outputEncoder: Encoder[Double] = Encoders.scalaDouble
}

val ds = spark.read.json("examples/src/main/resources/employees.json").as[Employee]
ds.show()
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

// Convert the function to a `TypedColumn` and give it a name
val averageSalary = MyAverage.toColumn.name("average_salary")
val result = ds.select(averageSalary)
result.show()
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedTypedAggregation.scala" </small>

```Java
import java.io.Serializable;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.TypedColumn;
import org.apache.spark.sql.expressions.Aggregator;

public static class Employee implements Serializable {
  private String name;
  private long salary;

  // Constructors, getters, setters...

}

public static class Average implements Serializable  {
  private long sum;
  private long count;

  // Constructors, getters, setters...

}

public static class MyAverage extends Aggregator<Employee, Average, Double> {
  // A zero value for this aggregation. Should satisfy the property that any b + zero = b
  public Average zero() {
    return new Average(0L, 0L);
  }
  // Combine two values to produce a new value. For performance, the function may modify `buffer`
  // and return it instead of constructing a new object
  public Average reduce(Average buffer, Employee employee) {
    long newSum = buffer.getSum() + employee.getSalary();
    long newCount = buffer.getCount() + 1;
    buffer.setSum(newSum);
    buffer.setCount(newCount);
    return buffer;
  }
  // Merge two intermediate values
  public Average merge(Average b1, Average b2) {
    long mergedSum = b1.getSum() + b2.getSum();
    long mergedCount = b1.getCount() + b2.getCount();
    b1.setSum(mergedSum);
    b1.setCount(mergedCount);
    return b1;
  }
  // Transform the output of the reduction
  public Double finish(Average reduction) {
    return ((double) reduction.getSum()) / reduction.getCount();
  }
  // Specifies the Encoder for the intermediate value type
  public Encoder<Average> bufferEncoder() {
    return Encoders.bean(Average.class);
  }
  // Specifies the Encoder for the final output value type
  public Encoder<Double> outputEncoder() {
    return Encoders.DOUBLE();
  }
}

Encoder<Employee> employeeEncoder = Encoders.bean(Employee.class);
String path = "examples/src/main/resources/employees.json";
Dataset<Employee> ds = spark.read().json(path).as(employeeEncoder);
ds.show();
// +-------+------+
// |   name|salary|
// +-------+------+
// |Michael|  3000|
// |   Andy|  4500|
// | Justin|  3500|
// |  Berta|  4000|
// +-------+------+

MyAverage myAverage = new MyAverage();
// Convert the function to a `TypedColumn` and give it a name
TypedColumn<Employee, Double> averageSalary = myAverage.toColumn().name("average_salary");
Dataset<Double> result = ds.select(averageSalary);
result.show();
// +--------------+
// |average_salary|
// +--------------+
// |        3750.0|
// +--------------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/java/org/apache/spark/examples/sql/JavaUserDefinedTypedAggregation.java"</small>

# 数据源

`Spark SQL`支持通过`DataFrame`接口对各种数据源进行操作。`DataFrame`可以使用 `relational transformations`操作，也可用于创建临时视图。将 `DataFrame`注册为临时视图允许您对其数据运行`SQL`查询。本节描述了使用`Spark Data Source` 加载和保存数据的一般方法，然后涉及可用于内置数据源的指定选项。

## 通用加载/保存方法

在最简单的形式中，默认数据源（`parquet`，除非另有配置 `spark.sql.sources.default`）将用于所有操作。

```Python
df = spark.read.load("examples/src/main/resources/users.parquet")
df.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py" </small>


### 手动指定选项

您还可以手动指定将与任何你想传递给数据源的其他选项一起使用的数据源。 数据源由其完全限定名称指定（即 `org.apache.spark.sql.parquet`），但是对于内置的源，你也可以使用它们的短名称（`json`，`parquet`，`jdbc`，`orc`，`libsvm`，`csv`，`text`）。从任何数据源类型加载的`DataFrame`都可以使用此语法转换为其他类型。

```Python
df = spark.read.load("examples/src/main/resources/people.json", format="json")
df.select("name", "age").write.save("namesAndAges.parquet", format="parquet")
```

<small>可以在`Spark`仓库找到完整例子examples/src/main/python/sql/datasource.py"</small>

要加载`CSV`文件，您可以使用：
```Python
df = spark.read.load("examples/src/main/resources/people.csv",
                     format="csv", sep=":", inferSchema="true", header="true")
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

在写操作期间也使用额外选项。 例如，您可以控制`ORC`数据源的`bloom`过滤器和字典编码。 以下`ORC`示例将在`favorite_color`上创建`bloom`过滤器，并对`name`和`favorite_color`使用字典编码。 对于`Parquet`，也存在`parquet.enable.dictionary`。 要查找有关额外`ORC/Parquet`选项的更多详细信息，请访问官方`Apache ORC/Parquet`网站。

```Python
df = spark.read.orc("examples/src/main/resources/users.orc")
(df.write.format("orc")
    .option("orc.bloom.filter.columns", "favorite_color")
    .option("orc.dictionary.key.threshold", "1.0")
    .save("users_with_options.orc"))
```
<small>可以在Spark仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

### 直接在文件上运行SQL

不使用读取`API`将文件加载到`DataFrame`并进行查询，也可以直接用`SQL`查询该文件.

```Python
df = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

### 保存模式

保存操作可以选择使用 `SaveMode`，它指定如何处理现有数据如果存在的话。重要的是要意识到，这些保存模式不使用任何锁定并且不是`atomic`（原子）的。另外，当执行 `Overwrite` 时，数据将在新数据写出之前被删除。

| Scala/Java | Any Language | Meaning |
| --- | --- | --- |
| `SaveMode.ErrorIfExists` (default) | `"error"` (default) | 将`DataFrame `保存到数据源时，如果数据已经存在，则会抛出异常。
| `SaveMode.Append` | `"append"` | 将 `DataFrame`保存到数据源时，如果 `data/table` 已存在，则 `DataFrame`的内容将被 `append`（附加）到现有数据中。
| `SaveMode.Overwrite` | `"overwrite"` | 覆盖模式意味着将`DataFrame`保存到数据源时，如果`data/table`已经存在，则预期`DataFrame`的内容将 覆盖现有数据。
| `SaveMode.Ignore` | `"ignore"` | 忽略模式意味着当将`DataFrame`保存到数据源时，如果数据已经存在，则保存操作预期不会保存`DataFrame` 的内容，并且不更改现有数据。这与 `SQL` 中的 `CREATE TABLE IF NOT EXISTS` 类似。

### 保存到持久化表

`DataFrames` 也可以使用 `saveAsTable` 命令作为持久保存到`Hive metastore`中。请注意，现有的`Hive`部署不需要使用此功能。`Spark`将为您创建默认的本地`Hive metastore`（使用`Derby`）。与 `createOrReplaceTempView` 命令不同，`saveAsTable` 将实现`DataFrame`的内容，并创建一个指向 `Hive metastore`中数据的指针。即使您的 `Spark`程序重新启动，持久性表仍然存在，因为您保持与同一个`metastore`的连接。可以通过使用表的名称在 `SparkSession` 上调用 `table` 方法来创建持久表的`DataFrame`。

对于基于文件的数据源，例如 `text`，`parquet`，`json`等，您可以通过 `path` 选项指定自定义表路），例如 `df.write.option("path", "/some/path").saveAsTable("t")`。当表被删除时，自定义表路径将不会被删除，并且表数据仍然存在。如果未指定自定义表路径，`Spark`将把数据写入仓库目录下的默认表路径。当表被删除时，默认的表路径也将被删除。

从`Spark 2.1`开始，持久性数据源表将每个分区元数据存储在`Hive metastore`中。这带来了几个好处:

*   由于`metastore`只能返回查询的必要分区，因此不再需要将第一个查询上的所有 `partitions discovering` 到表中。
*   `Hive DDL`s 如 `ALTER TABLE PARTITION ... SET LOCATION` 现在可用于使用 `Datasource API`创建的表。

请注意，创建外部数据源表（带有 `path` 选项）的表时，默认情况下不会收集分区信息。要 `sync`（同步）`metastore`中的分区信息，可以调用 `MSCK REPAIR TABLE`。

### 分桶，排序和分区

对于基于文件的数据源，也可以对输出进行 `bucket` 和 `sort` 或者 `partition`。`Bucketing` 和 `sorting` 仅适用于持久表
```Python
df.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

在使用 `Dataset API`时，分区可以同时与 `save` 和 `saveAsTable` 一起使用.

```Python
df.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

可以为单个表使用`partitioning`和`bucketing`:

```Python
df = spark.read.parquet("examples/src/main/resources/users.parquet")
(df
    .write
    .partitionBy("favorite_color")
    .bucketBy(42, "name")
    .saveAsTable("people_partitioned_bucketed"))
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

`partitionBy` 创建一个目录，如[分区发现](#分区发现) 部分所述。因此，对基数较高的 `columns` 的适用性有限。相反，`bucketBy` 可以在固定数量的`buckets` 中分配数据，并且可以在多个唯一值无界时使用数据。

## Parquet文件

[`Parquet`](http://parquet.io) 是许多其他数据处理系统支持的柱状格式。`Spark SQL`支持读写` Parquet` 文件，可自动保留原始数据的模式。当编写`Parquet`文件时，出于兼容性原因，所有`columns`都将自动转换为可空。

### 以编程的方式加载数据

使用上面例子中的数据:

```Python
peopleDF = spark.read.json("examples/src/main/resources/people.json")

# DataFrames can be saved as Parquet files, maintaining the schema information.
peopleDF.write.parquet("people.parquet")

# Read in the Parquet file created above.
# Parquet files are self-describing so the schema is preserved.
# The result of loading a parquet file is also a DataFrame.
parquetFile = spark.read.parquet("people.parquet")

# Parquet files can also be used to create a temporary view and then used in SQL statements.
parquetFile.createOrReplaceTempView("parquetFile")
teenagers = spark.sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
teenagers.show()
# +------+
# |  name|
# +------+
# |Justin|
# +------+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py"</small>

### 分区发现

表分区是在像`Hive`这样的系统中使用的常见的优化方法。在分区表中，数据通常存储在不同的目录中，分区列值编码在每个分区目录的路径中。`Parquet`数据源现在可以自动发现和推断分区信息。例如，我们可以使用以下目录结构将所有以前使用的人口数据存储到分区表中，其中有两个额外的列 `gender` 和 `country` 作为分区列:

```
path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...
```



通过将 `path/to/table` 传递给 `SparkSession.read.parquet` 或 `SparkSession.read.load`，`Spark SQL`将自动从路径中提取分区信息。现在返回的`DataFrame`的`schema`变成:

```
root
|-- name: string (nullable = true)
|-- age: long (nullable = true)
|-- gender: string (nullable = true)
|-- country: string (nullable = true)
```

请注意，会自动推断分区列的数据类型。目前，支持数字数据类型和字符串类型。有些用户可能不想自动推断分区列的数据类型。对于这些用例，自动类型推断可以由 `spark.sql.sources.partitionColumnTypeInference.enabled` 配置，默认为 `true`。当禁用类型推断时，字符串类型将用于分区列。

从`Spark 1.6.0`开始，默认情况下，分区发现只能找到给定路径下的分区。对于上述示例，如果用户将 `path/to/table/gender=male` 传递给 `SparkSession.read.parquet` 或 `SparkSession.read.load`，则 `gender` 将不被视为分区列。如果用户需要指定分区发现应该开始的基本路径，则可以在数据源选项中设置 `basePath`。例如，当 `path/to/table/gender=male` 是数据的路径并且用户将 `basePath` 设置为 `path/to/table/`，`gender` 将是一个分区列。

### 模式合并

像`ProtocolBuffer`，`Avro` 和`Thrift`一样，`Parquet`也支持模式演进。用户可以从一个简单的`schema`开始，并根据需要逐渐向 `schema` 添加更多的`columns`。以这种方式，用户可能会使用不同但相互兼容的多个`Parquet`文件的`schemas`。 `Parquet` 数据源现在能够自动检测这种情况并合并所有这些文件的`schemas`。

由于模式合并是一个相对昂贵的操作，并且在大多数情况下不是必需的，所以默认情况下从`1.5.0`开始。你可以按照如下的方式启用它:

1.  读取`Parquet`文件时，将数据源选项`mergeSchema` 设置为 `true`（如下面的例子所示），或
2.  将全局`SQL`选项`spark.sql.parquet.mergeSchema` 设置为 `true`。

```Python
from pyspark.sql import Row

# spark is from the previous example.
# Create a simple DataFrame, stored into a partition directory
sc = spark.sparkContext

squaresDF = spark.createDataFrame(sc.parallelize(range(1, 6))
                                  .map(lambda i: Row(single=i, double=i ** 2)))
squaresDF.write.parquet("data/test_table/key=1")

# Create another DataFrame in a new partition directory,
# adding a new column and dropping an existing column
cubesDF = spark.createDataFrame(sc.parallelize(range(6, 11))
                                .map(lambda i: Row(single=i, triple=i ** 3)))
cubesDF.write.parquet("data/test_table/key=2")

# Read the partitioned table
mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
mergedDF.printSchema()

# The final schema consists of all 3 columns in the Parquet files together
# with the partitioning column appeared in the partition directory paths.
# root
#  |-- double: long (nullable = true)
#  |-- single: long (nullable = true)
#  |-- triple: long (nullable = true)
#  |-- key: integer (nullable = true)
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py" </small>


### Hive-Metastore-Parquet表转换

当读取和写入`Hive metastore Parquet`表时，`Spark SQL`将尝试使用自己的`Parquet`支持，而不是`Hive SerDe`来获得更好的性能。此行为由 `spark.sql.hive.convertMetastoreParquet` 配置控制，默认情况下打开。

#### Hive/Parquet-Schema协调

从表格模式处理的角度来说，`Hive`和`Parquet`之间有两个关键的区别。

1.  `Hive`不区分大小写，而`Parquet`不是
2.  `Hive`认为所有`columns`列都可以为空，而`Parquet`中的可空性是重要的。

由于这个原因，当将`Hive metastore Parquet`表转换为`Spark SQL Parquet`表时，我们必须调整`metastore schema`与`Parquet schema`。`reconciliation` 规则是:

1.  在两个`schema`中具有相同名称的`Fields`必须具有相同的数据类型，而不管可空性。`reconciled field`应具有`Parquet`的数据类型，以便可空性得到尊重。

2.  `reconciled schema`正好包含`Hive metastore schema`中定义的那些字段。

    *   只出现在`Parquet schema`中的任何字段将被在`reconciled schema`中删除。
    *   仅在`Hive metastore schema`中出现的任何字段在 `reconciled schema` 中作为 `nullable field`添加。

#### 元数据刷新

`Spark SQL` 缓存`Parquet metadata`以获得更好的性能。当启用 `Hive metastore Parquet table`转换时，这些转换表的元数据也被缓存。如果这些表由`Hive`或其他外部工具更新，则需要手动刷新以确保一致的元数据。

```Python
# spark is an existing SparkSession
spark.catalog.refreshTable("my_table")
```

### 配置

可以使用 `SparkSession` 上的 `setConf` 方法或使用`SQL`运行 `SET key = value`命令来完成`Parquet`的配置.

| Property Name（参数名称）| Default（默认）| Meaning（含义）|
| --- | --- | --- |
| `spark.sql.parquet.binaryAsString` | false | 一些其他`Parquet`生产系统，特别是`Impala`，`Hive`和旧版本的`Spark SQL`，在写出`Parquet schema`时，不区分二进制数据和字符串。该`flag`告诉`Spark SQL`将二进制数据解释为`string`以提供与这些系统的兼容性。
| `spark.sql.parquet.int96AsTimestamp` | true | 一些`Parquet`生产系统，特别是`Impala`和`Hive`，将`Timestamp`存入`INT96`。该`flag`告诉`Spark SQL`将`INT96` 数据解析为`timestamp`以提供与这些系统的兼容性。
| `spark.sql.parquet.cacheMetadata` | true | 打开`Parquet schema metadata`的缓存。可以加快查询静态数据。
| `spark.sql.parquet.compression.codec` | snappy | 在编写`Parquet`文件时设置压缩编解码器的使用。可接受的值包括：`uncompressed`，`snappy`，`gzip`，`lzo`。
| `spark.sql.parquet.filterPushdown` | true | 设置为`true`时启用`Parquet filter push-down optimization`。
| `spark.sql.hive.convertMetastoreParquet` | true | 当设置为`false`时，`Spark SQL`将使用`Hive SerDe`作为`parquet tables`，而不是内置的支持。
| `spark.sql.parquet.mergeSchema` | false | 当为`true`时，`Parquet`数据源合并从所有数据文件收集的`schemas`，否则如果没有可用的`summary file`，则从`summary file`或`random data file`中挑选`schema`。
| `spark.sql.optimizer.metadataOnly` | true | 如果为`true`，则启用使用表的`metadata`的`metadata-only query optimization`来生成`partition columns`（分区列）而不是`table scans`表扫描）。当扫描的所有`columns`（列）都是`partition columns`分区列）并且`query`（查询）具有满足`distinct semantics`（不同语义）的`aggregate operator`（聚合运算符）时，它将适用。

## ORC文件
从`Spark 2.3`开始，`Spark`支持带有`ORC`文件的新`ORC`文件格式的矢量化`ORC`阅读器。 为此，新添加了以下配置。 当`spark.sql.orc.impl`设置为`native`并且`spark.sql.orc.enableVectorizedReader`设置为`true`时，矢量化读取器用于本机`ORC`表（例如，使用`USING ORC`子句创建的表）。 对于`Hive ORC serde`表（例如，使用`USING HIVE OPTIONS（fileFormat'ORC'`）子句创建的表），当`spark.sql.hive.convertMetastoreOrc`也设置为`true`时，使用矢量化阅读器。

| 属性名称 | 默认值 | 含义 | 
| --- | --- | --- |
| `spark.sql.orc.impl` | `native` | `ORC`实现的名称。 它可以是`native`和`hive`之一。`native`表示在`Apache ORC 1.4`上构建的本机`ORC`支持。 `hive`表示`Hive 1.2.1`中的`OR`C库。 |
| `spark.sql.orc.enableVectorizedReader` | `true` | 在本机实现中启用矢量化`ORC`解码。 如果为`false`，则在本机实现中使用新的非向量化`ORC`读取器。 对于`hive`实现，这将被忽略。 |

## JSON数据集

`Spark SQL`可以自动推断`JSON`数据集的`schema`，并将其作为`DataFrame`加载。可以使用`JSON`文件中的 `SparkSession.read.json` 进行此转换。

请注意，以`json`文件形式提供的文件不是典型的`JSON`文件。每行必须包含一个单独的，独立的有效的`JSON` 对象。有关更多信息，请参阅 [`JSON Lines`文本格式，也称为`newline`分隔的`JSON`](http://jsonlines.org/)

对于常规的多行`JSON`文件，将 `multiLine` 选项设置为 `true`。

```Python
# spark is from the previous example.
sc = spark.sparkContext

# A JSON dataset is pointed to by path.
# The path can be either a single text file or a directory storing text files
path = "examples/src/main/resources/people.json"
peopleDF = spark.read.json(path)

# The inferred schema can be visualized using the printSchema() method
peopleDF.printSchema()
# root
#  |-- age: long (nullable = true)
#  |-- name: string (nullable = true)

# Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

# SQL statements can be run by using the sql methods provided by sparkPython
teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
teenagerNamesDF.show()
# +------+
# |  name|
# +------+
# |Justin|
# +------+

# Alternatively, a DataFrame can be created for a JSON dataset represented by
# an RDD[String] storing one JSON object per string
jsonStrings = ['{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}']
otherPeopleRDD = sc.parallelize(jsonStrings)
otherPeople = spark.read.json(otherPeopleRDD)
otherPeople.show()
# +---------------+----+
# |        address|name|
# +---------------+----+
# |[Columbus,Ohio]| Yin|
# +---------------+----+
```

<small>可以在`Spark`仓库找到完整例子"examples/src/main/python/sql/datasource.py" </small>

## Hive表

`Spark SQL`还支持读取和写入存储在 [`Apache Hive`](http://hive.apache.org/) 中的数据。但是，由于`Hive`具有大量依赖关系，因此这些依赖关系不包含在默认`Spark` 分发中。如果在类路径中找到`Hive`依赖项，`Spark`将自动加载它们。请注意，这些`Hive`依赖关系也必须存在于所有工作节点上，因为它们将需要访问`Hive`序列化和反序列化库，以访问存储在`Hive`中的数据。

通过将 `hive-site.xml`，`core-site.xml`（用于安全配置）和 `hdfs-site.xml`（用于HDFS配置）文件放在 `conf/` 中来完成配置。

当使用`Hive`时，必须用`Hive`支持实例化 `SparkSession`，包括连接到持续的`Hive`转移，支持`Hive serdes`和`Hive`用户定义的功能。没有现有`Hive` 部署的用户仍然可以启用`Hive`支持。当 `hive-site.xml` 未配置时，上下文会自动在当前目录中创建 `metastore_db`，并创建由 `spark.sql.warehouse.dir` 配置的目录，该目录默认为`Spark`应用程序当前目录中的 `spark-warehouse` 目录 开始了 请注意，自从2.0.0以来，`hive-site.xml` 中的 `hive.metastore.warehouse.dir` 属性已被弃用。而是使用 `spark.sql.warehouse.dir` 来指定仓库中数据库的默认位置。您可能需要向启动`Spark`应用程序的用户授予写权限。

```Python
from os.path import expanduser, join, abspath

from pyspark.sql import SparkSession
from pyspark.sql import Row

# warehouse_location points to the default location for managed databases and tables
warehouse_location = abspath('spark-warehouse')

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL Hive integration example") \
    .config("spark.sql.warehouse.dir", warehouse_location) \
    .enableHiveSupport() \
    .getOrCreate()

# spark is an existing SparkSession
spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING) USING hive")
spark.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

# Queries are expressed in HiveQL
spark.sql("SELECT * FROM src").show()
# +---+-------+
# |key|  value|
# +---+-------+
# |238|val_238|
# | 86| val_86|
# |311|val_311|
# ...

# Aggregation queries are also supported.
spark.sql("SELECT COUNT(*) FROM src").show()
# +--------+
# |count(1)|
# +--------+
# |    500 |
# +--------+

# The results of SQL queries are themselves DataFrames and support all normal functions.
sqlDF = spark.sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key")

# The items in DataFrames are of type Row, which allows you to access each column by ordinal.
stringsDS = sqlDF.rdd.map(lambda row: "Key: %d, Value: %s" % (row.key, row.value))
for record in stringsDS.collect():
    print(record)
# Key: 0, Value: val_0
# Key: 0, Value: val_0
# Key: 0, Value: val_0
# ...

# You can also use DataFrames to create temporary views within a SparkSession.
Record = Row("key", "value")
recordsDF = spark.createDataFrame([Record(i, "val_" + str(i)) for i in range(1, 101)])
recordsDF.createOrReplaceTempView("records")

# Queries can then join DataFrame data with data stored in Hive.
spark.sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show()
# +---+------+---+------+
# |key| value|key| value|
# +---+------+---+------+
# |  2| val_2|  2| val_2|
# |  4| val_4|  4| val_4|
# |  5| val_5|  5| val_5|
# ...
```

<small>可以在Spark仓库找到完整例子"examples/src/main/python/sql/hive.py" </small>

### 指定Hive表的存储格式

创建`Hive`表时，需要定义如何从/向文件系统`read/write`数据，即 "输入格式"和 "输出格式"。您还需要定义该表如何将数据反序列化为行，或将行序列化为数据，即 "serde"。以下选项可用于指定存储格式（"serde", "input format", "output format"），例如，`CREATE TABLE src(id int) USING hive OPTIONS(fileFormat 'parquet')`。默认情况下，我们将以纯文本形式读取表格文件。请注意，`Hive`存储处理程序在创建表时不受支持，您可以使用`Hive`端的存储处理程序创建一个表，并使用`Spark SQL` 来读取它。

| 属性名字 | 含义 |
| --- | --- |
| `fileFormat` | `fileFormat`是一种存储格式规范的包，包括 "serde"，"input format" 和 "output format"。目前我们支持6个文件格式：'sequencefile'，'rcfile'，'orc'，'parquet'，'textfile'和'avro'。 |
| `inputFormat, outputFormat` | 这两个选项将相应的 "InputFormat" 和 "OutputFormat" 类的名称指定为字符串文字，例如：`org.apache.hadoop.hive.ql.io.orc.OrcInputFormat`。这两个选项必须成对出现，如果您已经指定了 "fileFormat" 选项，则无法指定它们。 |
| `serde` | 此选项指定 serde 类的名称。当指定 `fileFormat` 选项时，如果给定的 `fileFormat` 已经包含`serde`的信息，那么不要指定这个选项。目前的 "sequencefile"，"textfile" 和 "rcfile" 不包含`serde`信息，你可以使用这3个文件格式的这个选项。 |
| `fieldDelim, escapeDelim, collectionDelim, mapkeyDelim, lineDelim` | 这些选项只能与 "textfile" 文件格式一起使用。它们定义如何将分隔的文件读入行。 |

使用 `OPTIONS` 定义的所有其他属性将被视为`Hive serde`属性。

### 与不同版本的Hive-Metastore进行交互

`Spark SQL`的`Hive`支持的最重要的部分之一是与`Hive metastore`进行交互，这使得`Spark SQL`能够访问`Hive`表的元数据。从`Spark 1.4.0`开始，使用`Spark SQL` 的单一二进制构建可以使用下面所述的配置来查询不同版本的`Hive`转移。请注意，独立于用于与转移点通信的`Hive`版本，内部`Spark SQL`将针对`Hive 1.2.1` 进行编译，并使用这些类进行内部执行（`serdes`，`UDF`，`UDAF`等）。

以下选项可用于配置用于检索元数据的`Hive`版本：

| 属性名称 | 默认值 | 含义 |
| --- | --- | --- |
| `spark.sql.hive.metastore.version` | `1.2.1` | `Hive metastore`版本。可用选项为 `0.12.0` 至 `2.3.3`。 |
| `spark.sql.hive.metastore.jars` | `builtin` | 应用于实例化`HiveMetastoreClient`的`jar`的位置。该属性可以是三个选项之一: 1. `builtin`:当启用 `-Phive` 时，使用 `Hive 1.2.1`，它与`Spark`程序集捆绑在一起。选择此选项时，`spark.sql.hive.metastore.version`必须为 `1.2.1` 或未定义;2. `maven`:使用从`Maven`存储库下载的指定版本的`Hive jar`。通常不建议在生产部署中使用此配置;3. `JVM`的标准格式的`classpath`。该类路径必须包含所有`Hive`及其依赖项，包括正确版本的`Hadoop`。这些罐只需要存在于`driver`程序中，但如果您正在运行在`yarn` 集群模式，那么您必须确保它们与应用程序一起打包。|
| `spark.sql.hive.metastore.sharedPrefixes` | `com.mysql.jdbc,org.postgresql,com.microsoft.sqlserver,oracle.jdbc` | 使用逗号分隔的类前缀列表，应使用在 `Spark SQL`和特定版本的`Hive`之间共享的类加载器来加载。一个共享类的示例就是用来访问`Hive metastore`的`JDBC driver`。其它需要共享的类，是需要与已经共享的类进行交互的。例如，`log4j`使用的自定义`appender`。 |
| `spark.sql.hive.metastore.barrierPrefixes` | `(empty)` | 一个逗号分隔的类前缀列表，应该明确地为`Spark SQL`正在通信的`Hive` 的每个版本重新加载。例如，在通常将被共享的前缀中声明的`Hive UDF`（即：`org.apache.spark.*`）。 |

## JDBC连接其它数据库

`Spark SQL`还包括可以使用`JDBC`从其他数据库读取数据的数据源。此功能应优于使用 [`JdbcRDD`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.JdbcRDD)。这是因为结果作为`DataFrame`返回，并且可以轻松地在`Spark SQL`中处理或与其他数据源连接。`JDBC`数据源也更容易从`Java`或`Python`使用，因为它不需要用户提供`ClassTag`。（请注意，这不同于`Spark SQL JDBC`服务器，允许其他应用程序使用`Spark SQL`运行查询）。

要开始使用，您需要在`Spark`类路径中包含特定数据库的`JDBC driver`程序。例如，要从`Spark Shell`连接到`postgres`，您将运行以下命令:

```bash
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
```

可以使用`Data Sources API`将来自远程数据库的表作为`DataFrame`或`Spark SQL`临时视图进行加载。用户可以在数据源选项中指定`JDBC`连接属性。`用户` 和 `密码`通常作为登录数据源的连接属性提供。除了连接属性外，`Spark`还支持以下不区分大小写的选项:

| 属性名称 | 含义 |
| --- | --- |
| `url` | 要连接的`JDBC URL`。源特定的连接属性可以在`URL`中指定。例如`jdbc`：`jdbc:postgresql://localhost/test?user=fred&password=secret` |
| `dbtable` | 应该读取的`JDBC`表。请注意，可以使用在`SQL`查询的 `FROM` 子句中有效的任何内容。例如，您可以使用括号中的子查询代替完整表。 |
| `driver` | 用于连接到此`URL`的`JDBC driver`程序的类名。 |
| `partitionColumn, lowerBound, upperBound` | 如果指定了这些选项，则必须指定这些选项。另外，必须指定 `numPartitions`。他们描述如何从多个`worker` 并行读取数据时将表给分区。`partitionColumn` 必须是有问题的表中的数字列。请注意，`lowerBound` 和 `upperBound` 仅用于决定分区的大小，而不是用于过滤表中的行。因此，表中的所有行将被分区并返回。此选项仅适用于读操作。 |
| `numPartitions` | 在表读写中可以用于并行度的最大分区数。这也确定并发`JDBC`连接的最大数量。如果要写入的分区数超过此限制，则在写入之前通过调用 `coalesce(numPartitions)` 将其减少到此限制。 |
| `fetchsize` | `JDBC`抓取的大小，用于确定每次数据往返传递的行数。这有利于提升`JDBC driver`的性能，它们的默认值较小（例如：`Oracle`是10行）。该选项仅适用于读取操作。 |
| `batchsize` | `JDBC`批处理的大小，用于确定每次数据往返传递的行数。这有利于提升`JDBC driver`的性能。该选项仅适用于写操作。默认值为 `1000`。
| `isolationLevel` | 事务隔离级别，适用于当前连接。它可以是 `NONE`，`READ_COMMITTED`，`READ_UNCOMMITTED`，`REPEATABLE_READ`，或 `SERIALIZABLE` 之一，对应于`JDBC`连接对象定义的标准事务隔离级别，默认为 `READ_UNCOMMITTED`。此选项仅适用于写操作。请参考 `java.sql.Connection` 中的文档。 |
| `truncate` | 这是一个与`JDBC`相关的选项。启用 `SaveMode.Overwrite` 时，此选项会导致`Spark`截断现有表，而不是删除并重新创建。这可以更有效，并且防止表元数据（例如，索引）被移除。但是，在某些情况下，例如当新数据具有不同的模式时，它将无法工作。它默认为 `false`。此选项仅适用于写操作。 |
| `createTableOptions` | 这是一个与`JDBC`相关的选项。如果指定，此选项允许在创建表时设置特定于数据库的表和分区选项（例如：`CREATE TABLE t (name string) ENGINE=InnoDB.`）。此选项仅适用于写操作。 |
| `createTableColumnTypes` | 使用数据库列数据类型而不是默认值，创建表时。数据类型信息应以与 CREATE TABLE 列语法相同的格式指定（例如：`"name CHAR(64), comments VARCHAR(1024)"`）。指定的类型应该是有效的`spark sql`数据类型。此选项仅适用于写操作。 |

```Python
# Note: JDBC loading and saving can be achieved via either the load/save or jdbc methods
# Loading data from a JDBC source
jdbcDF = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql:dbserver") \
    .option("dbtable", "schema.tablename") \
    .option("user", "username") \
    .option("password", "password") \
    .load()

jdbcDF2 = spark.read \
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
          properties={"user": "username", "password": "password"})

# Saving data to a JDBC source
jdbcDF.write \
    .format("jdbc") \
    .option("url", "jdbc:postgresql:dbserver") \
    .option("dbtable", "schema.tablename") \
    .option("user", "username") \
    .option("password", "password") \
    .save()

jdbcDF2.write \
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
          properties={"user": "username", "password": "password"})

# Specifying create table column data types on write
jdbcDF.write \
    .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)") \
    .jdbc("jdbc:postgresql:dbserver", "schema.tablename",
          properties={"user": "username", "password": "password"})
```

<small>可以在Spark仓库找到完整例子"examples/src/main/python/sql/datasource.py" </small>

## 故障排除

*   `JDBC driver`程序类必须对客户端会话和所有执行程序上的原始类加载器可见。这是因为`Java`的`DriverManager`类执行安全检查，导致它忽略原始类加载器不可见的所有`driver`程序，当打开连接时。一个方便的方法是修改所有工作节点上的`compute_classpath.sh`以包含您的`driver`程序`JAR`。
*   一些数据库，例如`H2`，将所有名称转换为大写。您需要使用大写字母来引用`Spark SQL`中的这些名称。
*  用户可以在数据源选项中指定特定于供应商的`JDBC`连接属性以进行特殊处理。 例如，`spark.read.format("jdbc".option("url",oracleJdbcUrl).option（"oracle.jdbc.mapDateToTimestamp","false").oracle.jdbc.mapDateToTimestamp`默认为`true`，用户通常需要禁用此标志以避免`Oracle`日期被解析为时间戳。 

# 性能调优

对于某些工作负载，可以通过缓存内存中的数据或打开一些实验选项来提高性能。

## 在内存中缓存数据

`Spark SQL`可以通过调用 `spark.catalog.cacheTable("tableName")` 或 `dataFrame.cache()` 来使用内存中的列格式来缓存表。然后，`Spark SQL` 将只扫描所需的列，并将自动调整压缩以最小化内存使用量和GC压力。您可以调用 `spark.catalog.uncacheTable("tableName")` 从内存中删除该表。

内存缓存的配置可以使用 `SparkSession` 上的 `setConf` 方法或使用`SQL`运行 `SET key=value` 命令来完成。

| 属性名称 | 默认 | 含义 |
| --- | --- | --- |
| `spark.sql.inMemoryColumnarStorage.compressed` | true | 当设置为`true`时，`Spark SQL`将根据数据的统计信息为每个列自动选择一个压缩编解码器。 |
| `spark.sql.inMemoryColumnarStorage.batchSize` | 10000 | 控制批量的柱状缓存的大小。更大的批量大小可以提高内存利用率和压缩率，但是在缓存数据时会冒出`OOM`风险。 |

## 其他配置选项

以下选项也可用于调整查询执行的性能。这些选项可能会在将来的版本中被废弃，因为更多的优化是自动执行的。

| 属性名称 | 默认值 | 含义 |
| --- | --- | --- |
| `spark.sql.files.maxPartitionBytes` | 134217728 (128 MB) | 在读取文件时，将单个分区打包的最大字节数。 |
| `spark.sql.files.openCostInBytes` | 4194304 (4 MB) | 按照字节数来衡量的打开文件的估计费用可以在同一时间进行扫描。将多个文件放入分区时使用。最好过度估计，那么具有小文件的分区将比具有较大文件的分区（首先计划的）更快。 |
| `spark.sql.broadcastTimeout` | 300 | 广播连接中的广播等待时间超时（秒）|
| `spark.sql.autoBroadcastJoinThreshold` | 10485760 (10 MB) | 配置执行连接时将广播给所有工作节点的表的最大大小（以字节为单位）。通过将此值设置为-1可以禁用广播。请注意，目前的统计信息仅支持`Hive Metastore`表，其中已运行命令 `ANALYZE TABLE <tableName&gt> COMPUTE STATISTICS noscan`。 |
| `spark.sql.shuffle.partitions` | 200 | 配置在为连接或聚合洗牌数据时要使用的分区数。

## SQL查询的广播提示
`BROADCAST`提示指导`Spark`在将其与另一个表或视图连接时广播每个指定的表。 当`Spark`决定连接方法时，广播散列连接（即`BHJ`）是首选，即使统计信息高于配置`spark.sql.autoBroadcastJoinThreshold`。 指定连接的两端时，`Spark`会广播具有较低统计信息的那一方。 注意`Spark`并不保证始终选择`BHJ`，因为并非所有情况（例如全外连接）都支持`BHJ`。 当选择广播嵌套循环连接时，我们仍然尊重提示。
```
from pyspark.sql.functions import broadcast
broadcast(spark.table("src")).join(spark.table("records"), "key").show()
```

# 分布式SQL引擎

`Spark SQL`也可以充当使用其`JDBC/ODBC`或命令行界面的分布式查询引擎。在这种模式下，最终用户或应用程序可以直接与`Spark SQL`交互运行`SQL`查询，而不需要编写任何代码。

## 运行Thrift-JDBC/ODBC服务器

这里实现的`Thrift JDBC/ODBC`服务器对应于`Hive 1.2`中的 [`HiveServer2`](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2)。您可以使用`Spark`或`Hive 1.2.1`附带的直线脚本测试`JDBC`服务器。

要启动`JDBC/ODBC`服务器，请在`Spark`目录中运行以下命令:

```bash
./sbin/start-thriftserver.sh 
```

此脚本接受所有 `bin/spark-submit` 命令行选项，以及 `--hiveconf` 选项来指定 Hive 属性。您可以运行 `./sbin/start-thriftserver.sh --help` 查看所有可用选项的完整列表。默认情况下，服务器监听`localhost:10000`。您可以通过环境变量覆盖此行为，即:

```bash
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
```

或系统属性:

```bash
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```

现在，您可以使用`beeline`来测试`Thrift JDBC/ODBC`服务器:

```bash
./bin/beeline 
```

使用`beeline`方式连接到`JDBC/ODBC`服务器:

```bash
beeline> !connect jdbc:hive2://localhost:10000 
```

`Beeline`将要求您输入用户名和密码。在非安全模式下，只需输入机器上的用户名和空白密码即可。对于安全模式，请按照 [`beeline`文档](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients) 中的说明进行操作。

配置`Hive`是通过将 `hive-site.xml`, `core-site.xml` 和 `hdfs-site.xml` 文件放在 `conf/` 中完成的。

您也可以使用`Hive`附带的`beeline`脚本。

`Thrift JDBC`服务器还支持通过`HTTP`传输发送`thrift RPC`消息。使用以下设置启用`HTTP`模式作为系统属性或在 `conf/` 中的 `hive-site.xml` 文件中启用:

```bash
hive.server2.transport.mode - Set this to value: http
hive.server2.thrift.http.port - HTTP port number to listen on; default is 10001
hive.server2.http.endpoint - HTTP endpoint; default is cliservice 
```

要测试，请使用`beeline`以`http`模式连接到`JDBC/ODBC`服务器:

```bash
beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint> 
```

## 运行Spark-SQL-CLI

`Spark SQL CLI`是在本地模式下运行`Hive`转移服务并执行从命令行输入的查询的方便工具。请注意，`Spark SQL CLI`不能与`Thrift JDBC`服务器通信。

要启动`Spark SQL CLI`，请在`Spark`目录中运行以下命令:

```bash
./bin/spark-sql 
```

配置`Hive`是通过将 `hive-site.xml`，`core-site.xml` 和 `hdfs-site.xml` 文件放在 `conf/` 中完成的。您可以运行 `./bin/spark-sql --help` 获取所有可用选项的完整列表。

# 使用Apache-Arrow的Pandas-PySpark使用指南
## Spark中的Apache-Arrow
`Apache Arrow`是一种内存中的列式数据格式，在`Spark`中用于在`JVM`和`Python`进程之间有效地传输数据。 这对于使用`Pandas NumPy`数据的`Python`用户来说是最有益的。 它的使用不是自动的，可能需要对配置或代码进行一些小的更改才能充分利用并确保兼容性。 本指南将提供有关如何在`Spark`中使用`Arrow`的高级描述，并在使用启用箭头的数据时突出显示任何差异。

### 确保PyArrow已安装
如果使用`pip`安装`PySpark`，则可以使用命令`pip install pyspark [sql]`将`PyArrow`作为`SQL`模块的额外依赖项引入。 否则，您必须确保在所有群集节点上安装并可用`PyArrow`。 当前支持的版本是`0.8.0`。 您可以使用`conda-forge`通道中的`pip`或`conda`进行安装。 有关详细信息，请参阅`PyArrow`[安装](https://arrow.apache.org/docs/python/install.html)。

## 启用与Pandas的转换
使用调用`toPandas()`将`Spark DataFrame`转换为`Pandas DataFrame`时以及使用`createDataFrame(pandas_df)`从`Pandas DataFrame`创建`Spark DataFrame`时，`Arrow`可用作优化。 要在执行这些调用时使用`Arrow`，用户需要先将`Spark`配置`'spark.sql.execution.arrow.enabled'`设置为`'true'`。 默认情况下禁用此功能。

此外，如果在`Spark`中的实际计算之前发生错误，则由`'spark.sql.execution.arrow.enabled'`启用的优化可以自动回退到非Arrow优化实现。 这可以通过`'spark.sql.execution.arrow.fallback.enabled'`来控制。

```Python
import numpy as np
import pandas as pd

# Enable Arrow-based columnar data transfers
spark.conf.set("spark.sql.execution.arrow.enabled", "true")

# Generate a Pandas DataFrame
pdf = pd.DataFrame(np.random.rand(100, 3))

# Create a Spark DataFrame from a Pandas DataFrame using Arrow
df = spark.createDataFrame(pdf)

# Convert the Spark DataFrame back to a Pandas DataFrame using Arrow
result_pdf = df.select("*").toPandas()
```
完整的示例代码"examples/src/main/python/sql/arrow.py"。

使用上述箭头优化将产生与未启用箭头时相同的结果。 请注意，即使使用`Arrow`，`toPandas()`也会将`DataFrame`中所有记录收集到驱动程序中，并且应该在一小部分数据上完成。 当前不支持所有`Spark`数据类型，如果列具有不受支持的类型，则可能引发错误，请参阅支持的`SQL`类型。 如果在`createDataFrame()`期间发生错误，`Spark`将回退以创建没有`Arrow`的`DataFrame`。

## Pandas-UDF
`Pandas UD`F是用户定义的函数，由`Spark`使用`Arrow`执行传输数据和`Pandas`以处理数据。 `Pandas UDF`使用关键字`pandas_udf`作为装饰器定义或包装函数，不需要其他配置。 目前，有两种类型的`Pandas UDF：Scalar`和`Grouped Map`。

### 标量
标量`Pandas UDF`用于矢量化标量操作。 它们可以与`select`和`withColumn`等函数一起使用。 `Python`函数应该将`pandas.Series`作为输入并返回相同长度的`pandas.Series`。 在内部，`Spark`将执行`Pandas UDF`，方法是将列拆分为批次，并将每个批次的函数作为数据的子集调用，然后将结果连接在一起。

以下示例显示如何创建计算2列乘积的标量`Pandas UDF`。

```Python
import pandas as pd

from pyspark.sql.functions import col, pandas_udf
from pyspark.sql.types import LongType

# Declare the function and create the UDF
def multiply_func(a, b):
    return a * b

multiply = pandas_udf(multiply_func, returnType=LongType())

# The function for a pandas_udf should be able to execute with local Pandas data
x = pd.Series([1, 2, 3])
print(multiply_func(x, x))
# 0    1
# 1    4
# 2    9
# dtype: int64

# Create a Spark DataFrame, 'spark' is an existing SparkSession
df = spark.createDataFrame(pd.DataFrame(x, columns=["x"]))

# Execute function as a Spark vectorized UDF
df.select(multiply(col("x"), col("x"))).show()
# +-------------------+
# |multiply_func(x, x)|
# +-------------------+
# |                  1|
# |                  4|
# |                  9|
# +-------------------+
```

完整的示例代码"examples/src/main/python/sql/arrow.py"。

###  分组映射
分组映射`Pandas UDF`与`groupBy().apply()`一起使用，它实现了"split-apply-combine"模式。 `Split-apply-combine`包含三个步骤：
*  使用`DataFrame.groupBy`将数据拆分为组。
*  在每个组上应用一个功能。 该函数的输入和输出都是`pandas.DataFrame`。 输入数据包含每个组的所有行和列。
*  将结果合并到一个新的`DataFrame`中。

要使用`groupBy().apply()`，用户需要定义以下内容：
*  一个`Pytho`n函数，用于定义每个组的计算。
*  `StructType`对象或定义输出`DataFrame`架构的字符串。

如果指定为字符串，则返回的`pandas.DataFrame`的列标签必须与定义的输出模式中的字段名称匹配，或者如果不是字符串，则必须按字段匹配字段数据类型，例如， 整数指数。 有关如何在构造`pandas.DataFrame`时标记列的信息，请参阅`pandas.DataFrame`。

请注意，在应用函数之前，组的所有数据都将加载到内存中。 这可能导致内存不足异常，尤其是在组大小偏斜的情况下。 `maxRecordsPerBatch`的配置不适用于组，并且由用户决定分组数据是否适合可用内存。

以下示例显示如何使用`groupby().apply()`从组中的每个值中减去均值。

```Python
from pyspark.sql.functions import pandas_udf, PandasUDFType

df = spark.createDataFrame(
    [(1, 1.0), (1, 2.0), (2, 3.0), (2, 5.0), (2, 10.0)],
    ("id", "v"))

@pandas_udf("id long, v double", PandasUDFType.GROUPED_MAP)
def subtract_mean(pdf):
    # pdf is a pandas.DataFrame
    v = pdf.v
    return pdf.assign(v=v - v.mean())

df.groupby("id").apply(subtract_mean).show()
# +---+----+
# | id|   v|
# +---+----+
# |  1|-0.5|
# |  1| 0.5|
# |  2|-3.0|
# |  2|-1.0|
# |  2| 4.0|
# +---+----+
```

完整的示例代码"examples/src/main/python/sql/arrow.py"。

有关详细用法，请参阅[`pyspark.sql.functions.pandas_udf`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.functions.pandas_udf)和[`pyspark.sql.GroupedData.apply`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.GroupedData.apply)。

### 分组聚合
分组聚合`Pandas UDF`类似于`Spark`聚合函数。 分组聚合`Pandas UDF`与`groupBy().agg()`和`pyspark.sql.Window`一起使用。 它定义了从一个或多个`pandas.Series`到标量值的聚合，其中每个`pandas.Series`表示组或窗口中的列。

请注意，此类型的`UDF`不支持部分聚合，组或窗口的所有数据都将加载到内存中。 此外，目前只有`Grouped`聚合`Pandas UDF`支持无界窗口。

以下示例显示如何使用此类型的`UDF`来计算`groupBy`和窗口操作的平均值：
```Python
from pyspark.sql.functions import pandas_udf, PandasUDFType
from pyspark.sql import Window

df = spark.createDataFrame(
    [(1, 1.0), (1, 2.0), (2, 3.0), (2, 5.0), (2, 10.0)],
    ("id", "v"))

@pandas_udf("double", PandasUDFType.GROUPED_AGG)
def mean_udf(v):
    return v.mean()

df.groupby("id").agg(mean_udf(df['v'])).show()
# +---+-----------+
# | id|mean_udf(v)|
# +---+-----------+
# |  1|        1.5|
# |  2|        6.0|
# +---+-----------+

w = Window \
    .partitionBy('id') \
    .rowsBetween(Window.unboundedPreceding, Window.unboundedFollowing)
df.withColumn('mean_v', mean_udf(df['v']).over(w)).show()
# +---+----+------+
# | id|   v|mean_v|
# +---+----+------+
# |  1| 1.0|   1.5|
# |  1| 2.0|   1.5|
# |  2| 3.0|   6.0|
# |  2| 5.0|   6.0|
# |  2|10.0|   6.0|
# +---+----+------+
```
完整的示例代码"examples/src/main/python/sql/arrow.p"。

更详细的用法，参见[`pyspark.sql.functions.pandas_udf`](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.functions.pandas_udf)。

## 使用笔记
### 支持的SQL类型
目前，基于箭头的转换支持所有`Spark SQL`数据类型，但`MapType`，`TimestampType.ArrayType`和嵌套的`StructType`除外。仅当安装的`PyArrow`等于或高于`0.10.0`时才支持`BinaryType`。

### 设置箭头批量大小
`Spark`中的数据分区将转换为箭头记录批次，这可能会暂时导致`JVM`中的高内存使用量。为避免可能的内存不足异常，可以通过将`conf"spark.sql.execution.arrow.maxRecordsPerBatch"`设置为一个整数来调整箭头记录批次的大小，该整数将确定每个批次的最大行数。默认值为每批10,000条记录。如果列数很大，则应相应地调整该值。使用此限制，每个数据分区将被制成一个或多个记录批次以进行处理。

### 带时区语义的时间戳
`Spar`k在内部将时间戳存储为`UTC`值，并且在没有指定时区的情况下引入的时间戳数据将以本地时间转换为UTC，并具有微秒分辨率。在`Spark中`导出或显示时间戳数据时，会话时区用于本地化时间戳值。会话时区使用配置`'spark.sql.session.timeZone'`设置，如果未设置，将默认为`JVM`系统本地时区。 `Pandas`使用具有纳秒分辨率的`datetime64`类型，`datetime64 [ns]`，并且每列具有可选的时区。

当时间戳数据从`Spark`传输到`Pandas`时，它将转换为纳秒，每列将转换为`Spark`会话时区，然后本地化到该时区，这将删除时区并将值显示为本地时间。使用`timestamp`列调用`toPandas()`或`pandas_udf`时会发生这种情况。

当时间戳数据从`Pandas`传输到`Spark`时，它将转换为`UTC`微秒。当使用`Pandas DataFrame`调用`createDataFrame`或从`pandas_udf`返回时间戳时，会发生这种情况。这些转换是自动完成的，以确保`Spark`具有预期格式的数据，因此不必自己进行任何这些转换。任何纳秒值都将被截断。

请注意，标准`UDF`（非`Pandas`）会将时间戳数据作为`Python`日期时间对象加载，这与`Pandas`时间戳不同。在`pandas_udfs`中使用时间戳时，建议使用`Pandas`时间序列功能以获得最佳性能，有关详细信息，请参阅此处。

# 迁移指南

略

# 参考

## 数据类型

`Spark SQL`和`DataFrames`支持下面的数据类型:

*   N数值类型类型
    *   `ByteType`: 表示1个字节的有符号整数。数字范围是从-128到127。
    *   `ShortType`: 表示2个字节的有符号整数。数字范围是从-32768到32767。
    *   `IntegerType`: 表示4个字节的有符号整数。数字范围是从-2147483648到2147483647。
    *   `LongType`: 表示8字节有符号整数。数字范围是从-9223372036854775808到9223372036854775807。
    *   `FloatType`: 表示4字节单精度浮点数。
    *   `DoubleType`: 表示8字节双精度浮点数。
    *   `DecimalType`: 表示任意精度的带符号十进制数字。内部支持`java.math.BigDecimal`。A BigDecimal由任意精度的整数无标度值和32位整数标度组成。
*   字符串类型
    *   `StringType`: 表示字符串值
*   二进制类型
    *   `BinaryType`: 表示字节序列值。
*   布尔类型
    *   `BooleanType`: 表示布尔值
*   日期和时间类型
    *   `TimestampType`: 代表包含年，月，日，小时，分钟和秒的字段值的值。
    *   `DateType`: 代表包含年，月，日字段值的值。
*   复杂类型
    *   `ArrayType(elementType, containsNull)`：表示包含一系列类型为的元素的值`elementType`。`containsNull`用于指示`ArrayType`值中的元素是否可以具有`null`值。
    *   `MapType(keyType, valueType, valueContainsNull)`：表示包含一组键值对的值。键的数据类型用来描述，`keyType`而值的数据类型用来描述`valueType`。对于`MapType`值，键不允许具有`null`值。`valueContainsNull` 用于指示值的`MapType`值是否可以具有`null`值。
    *   `StructType(fields)`：表示具有由`StructFields(fields)`序列描述的结构的值。
    *   `StructField(name, dataType, nullable)`：表示中的字段`StructType`。字段的名称用表示`name`。字段的数据类型由表示`dataType`。`nullable`用于指示此字段的`null`值是否可以具有值。

`Spark SQL`的所有数据类型都在 `pyspark.sql.types` 的包中。你可以通过如下方式来访问它们.

```Python
from pyspark.sql.types import *
```

| 数据类型 | Python中的值类型 | 用于访问或创建数据类型的API |
| --- | --- | --- |
| **ByteType** | int or long
**Note:** 数字将在运行时转换为1字节有符号整数。请确保数字在-128到127的范围内。 | ByteType() |
| **ShortType** | int or long
**Note:** 数字将在运行时转换为2字节有符号整数。请确保数字在-32768到32767的范围内 | ShortType() |
| **IntegerType** | int or long | IntegerType() |
| **LongType** | long
**Note:** 数字将在运行时转换为8字节有符号整数。请确保数字在-9223372036854775808至9223372036854775807范围内。否则，请将数据转换为十进制。十进制并使用DecimalType | LongType() |
| **FloatType** | float
**Note:** 数字将在运行时转换为4字节单精度浮点数。 | FloatType() |
| **DoubleType** | float | DoubleType() |
| **DecimalType** | decimal.Decimal | DecimalType() |
| **StringType** | string | StringType() |
| **BinaryType** | bytearray | BinaryType() |
| **BooleanType** | bool | BooleanType() |
| **TimestampType** | datetime.datetime | TimestampType() |
| **DateType** | datetime.date | DateType() |
| **ArrayType** | 列表，元组或数组 | ArrayType(_elementType_, [_containsNull_])
**Note:** `containsNull`的默认值是真。 |
| **MapType** | dict | MapType(_keyType_, _valueType_, [_valueContainsNull_])
**Note:** `valueContainsNull`的默认值是真. |
| **StructType** | list or tuple | StructType(_fields_)
**Note:**  `fields`是`StructFields`的`Seq`。另外，不允许使用两个名称相同的字段。 |
| **StructField** | 此字段的数据类型在`Python`中的值类型（例如，对于数据类型为`IntegerType`的`StructField`为`Int` | StructField(_name_, _dataType_, [_nullable_])
**Note:** `nullable`的·默认值是真. |

| Data type | Value type in R | API to access or create a data type |
| --- | --- | --- |
| **ByteType** | integer
**Note:** Numbers will be converted to 1-byte signed integer numbers at runtime. Please make sure that numbers are within the range of -128 to 127. | "byte" |
| **ShortType** | integer
**Note:** Numbers will be converted to 2-byte signed integer numbers at runtime. Please make sure that numbers are within the range of -32768 to 32767. | "short" |
| **IntegerType** | integer | "integer" |
| **LongType** | integer
**Note:** Numbers will be converted to 8-byte signed integer numbers at runtime. Please make sure that numbers are within the range of -9223372036854775808 to 9223372036854775807. Otherwise, please convert data to decimal.Decimal and use DecimalType. | "long" |
| **FloatType** | numeric
**Note:** Numbers will be converted to 4-byte single-precision floating point numbers at runtime. | "float" |
| **DoubleType** | numeric | "double" |
| **DecimalType** | Not supported | Not supported |
| **StringType** | character | "string" |
| **BinaryType** | raw | "binary" |
| **BooleanType** | logical | "bool" |
| **TimestampType** | POSIXct | "timestamp" |
| **DateType** | Date | "date" |
| **ArrayType** | vector or list | list(type="array", elementType=_elementType_, containsNull=[_containsNull_])
**Note:** The default value of _containsNull_ is _TRUE_. |
| **MapType** | environment | list(type="map", keyType=_keyType_, valueType=_valueType_, valueContainsNull=[_valueContainsNull_])
**Note:** The default value of _valueContainsNull_ is _TRUE_. |
| **StructType** | named list | list(type="struct", fields=_fields_)
**Note:** _fields_ is a Seq of StructFields. Also, two fields with the same name are not allowed. |
| **StructField** | The value type in R of the data type of this field (For example, integer for a StructField with the data type IntegerType) | list(name=_name_, type=_dataType_, nullable=[_nullable_])
**Note:** The default value of _nullable_ is _TRUE_. |

## NaN语义学

当处理一些不符合标准浮点数语义的 `float` 或 `double` 类型时，对于 Not-a-Number(NaN) 需要做一些特殊处理。具体如下:

*   `NaN = NaN`返回`true`。
*   在聚合操作中，所有的`NaN`值将被分到同一个组中。
*   在`join` `key`中`NaN`可以当做一个普通的值。
*   `NaN`值在升序排序中排到最后，比任何其他数值都大。

## 算术运算
不检查对数字类型执行的操作（十进制除外）是否溢出。 这意味着如果操作导致溢出，结果与`Java/Scala`程序中相同操作返回的结果相同（例如，如果2个整数的总和高于可表示的最大值，则结果为负数）。