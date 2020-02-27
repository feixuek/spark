# Spark 配置

*   [Spark 属性](#spark-properties)
    *   [动态加载 Spark 属性](#动态加载-spark-属性)
    *   [查看 Spark 属性](#查看-spark-属性)
    *   [可用属性](#可用属性)
        *   [应用程序属性](#应用程序属性)
        *   [运行环境](#运行环境)
        *   [Shuffle 行为](#shuffle-behavior-shuffle-行为)
        *   [Spark UI](#spark-ui)
        *   [Compression and Serialization（压缩和序列化）](#compression-and-serialization-压缩和序列化)
        *   [Memory Management（内存管理）](#memory-management-内存管理)
        *   [Execution Behavior（执行行为）](#execution-behavior-执行行为)
        *   [Networking（网络）](#networking-网络)
        *   [Scheduling（调度）](#scheduling-调度)
        *   [Dynamic Allocation（动态分配）](#dynamic-allocation-动态分配)
        *   [Security（安全）](#security-安全)
        *   [TLS / SSL](#tls--ssl)
        *   [Spark SQL](#spark-sql)
        *   [Spark Streaming](#spark-streaming)
        *   [SparkR](#sparkr)
        *   [GraphX](#graphx)
        *   [Deploy（部署）](#deploy-部署)
        *   [Cluster Managers（集群管理器）](#cluster-managers-集群管理器)
            *   [](#yarn)[YARN](running-on-yarn.html#configuration)
            *   [](#mesos)[Mesos](running-on-mesos.html#configuration)
            *   [](#standalone-mode)[Standalone Mode](spark-standalone.html#cluster-launch-scripts)
*   [环境变量](#environment-variables)
*   [配置 Logging](#configuring-logging)
*   [Overriding configuration directory（覆盖配置目录）](#overriding-configuration-directory-覆盖配置目录)
*   [Inheriting Hadoop Cluster Configuration（继承 Hadoop 集群配置）](#inheriting-hadoop-cluster-configuration-继承-hadoop-集群配置)

Spark 提供了三个位置来配置系统:

*   [Spark 属性](#spark-properties) 控制着大多数应用参数，并且可以通过使用一个 [SparkConf](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf) 对象来设置，或者通过 Java 系统属性来设置.
*   [环境变量](#environment-variables) 可用于在每个节点上通过 `conf/spark-env.sh` 脚本来设置每台机器设置，例如`IP`地址.
*   [Logging](#configuring-logging) 可以通过 `log4j.properties` 来设置.

# Spark 属性

`Spark`属性控制大多数应用程序设置，并为每个应用程序单独配置。这些属性可以直接在 [SparkConf](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf) 上设置并传递给您的 `SparkContext`。`SparkConf` 可以让你配置一些常见的属性（例如`master URL`和应用程序名称），以及通过 `set()` 方法来配置任意键值对。例如，我们可以使用两个线程初始化一个应用程序，如下所示：

请注意，我们运行`local[2]`，意思是两个线程 - 代表 "最小"并行性，这可以帮助检测在只存在于分布式环境中运行时的错误.

```
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

注意，本地模式下，我们可以使用多个线程，而且在像`Spark Streaming`这样的场景下，我们可能需要多个线程来防止任一类型的类似线程饿死这样的问题。配置时间段的属性应该写明时间单位，如下格式都是可接受的:

```
25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years) 
```

指定字节大小的属性应该写明单位。如下格式都是可接受的：

```
1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes) 
```

虽然没有单位的数字通常被解释为字节，但有些数字被解释为`KiB`或`MiB`。 请参阅各个配置属性的文档。 尽可能指定单位。

## 动态加载 Spark 属性

在某些场景下，你可能想避免将属性值写死在`SparkConf`中。例如，你可能希望在同一个应用上使用不同的`master`或不同的内存总量。`Spark`允许你简单地创建一个空的`conf`:

```
val sc = new SparkContext(new SparkConf())
```

然后在运行时设置这些属性 :

```
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```

`Spark shell`和 [`spark-submit`](https://spark.apache.org/docs/latest/submitting-applications.html) 工具支持两种动态加载配置的方法。第一种，通过命令行选项，如：上面提到的 `--master`。`spark-submit` 可以使用 `--conf` 标志来接受任何`Spark`属性标志，但对于启动 `Spark`应用程序的属性使用特殊标志。运行 `./bin/spark-submit --help` 可以展示这些选项的完整列表.

`bin/spark-submit` 也支持从 `conf/spark-defaults.conf` 中读取配置选项，其中每行由一个键和一个由空格分隔的值组成，如下:

```
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer 
```

指定为标志或属性文件中的任何值都将传递给应用程序并与通过`SparkConf`指定的那些值合并。属性直接在`SparkConf`上设置采取最高优先级，然后标志传递给 `spark-submit` 或 `spark-shell`，然后选项在 `spark-defaults.conf` 文件中。自从`Spark`版本的早些时候，一些配置键已被重命名 ; 在这种情况下，旧的键名仍然被接受，但要比较新的键优先级都要低一些。

`Spark`属性主要可以分为两种：一种与`deploy`相关，如`spark.driver.memory`，`spark.executor.instances`，在运行时通过`SparkConf`以编程方式设置时，这种属性可能不受影响，或者 行为取决于您选择的集群管理器和部署模式，因此建议通过配置文件或`spark-submit`命令行选项进行设置; 另一个主要与`Spark`运行时控件有关，比如`spark.task.maxFailures`，这种属性可以以任何一种方式设置。

## 查看 Spark 属性

在应用程序的 web UI `http://<driver>;:4040` 中，`Environment`选项卡中列出了`Spark`的属性。这是一个检查您是否正确设置了您的属性的一个非常有用的地方。注意，只有显示地通过 `spark-defaults.conf`，`SparkConf` 或者命令行设置的值将会出现。对于所有其他配置属性，您可以认为使用的都是默认值.

## 可用属性

大多数控制内部设置的属性具有合理的默认值。一些常见的选项是：

### 应用程序属性

| 属性名称 | 默认值 | 含义 |
| --- | --- | --- |
| `spark.app.name` | (none) | `Spark`应用的名字。会在UI和日志中出现。 |
| `spark.driver.cores` | 1 | 在`cluster`模式下，用几个`core`运行`driver`进程。 |
| `spark.driver.maxResultSize` | 1g | `Spark action`算子返回的结果集的最大数量。至少要1M，可以设为 0 表示无限制。如果结果超过这一大小`Spark job` 会直接中断退出。但是，设得过高有可能导致`driver`出现`out-of-memory`异常（取决于`spark.driver.memory`设置，以及驱动器`JVM`的内存限制）。设一个合理的值，以避免 `driver` 出现 `out-of-memory` 异常。 |
| `spark.driver.memory` | 1g | `driver`进程可以使用的内存总量（例如：`1g`，`2g`）。注意，在`client`模式下，这个配置不能在`SparkConf`中直接设置，应为在那个时候 `driver`进程的`JVM`已经启动了。因此需要在命令行里用`--driver-memory`选项 或者在默认属性配置文件里设置。 |
| `spark.driver.memoryOverhead` | driverMemory * 0.10, 最小值384 | 除非另行指定，否则在群集模式下为每个驱动程序分配的堆外内存量（MiB）。 这是一个内存，可以解决诸如`JVM`开销，实习字符串，其他本地开销等问题。这会随着容器大小（通常为6-10％）而增长。 `YARN`和`Kubernetes`目前支持此选项。
| `spark.executor.memory` | 1g | 每个 executor 进程使用的内存总量（例如，`2g`，`8g`）。Amount of memory to use per executor process (例如，`2g`，`8g`). |
| `spark.executor.pyspark.memory` | Not set | 除非另有说明，否则`MiB`中每个执行程序中分配给`PySpark`的内存量。 如果设置，执行程序的`PySpark`内存将限制为此数量。 如果没有设置，`Spark`不会限制`Python`的内存使用，应由应用程序来避免超出与其他非`JVM`进程共享的开销内存空间。 当`PySpark`在`YARN`或`Kubernetes`中运行时，此内存将添加到执行程序资源请求中。 注意：`Python`内存使用可能不限于不支持资源限制的平台，例如`Windows`。 | 
| `spark.executor.memoryOverhead` | executorMemory * 0.10, 最小值384 | 除非另有说明，否则每个执行程序要分配的堆外内存量（MiB）。 这是一个内存，可以解决诸如`JVM`开销，实习字符串，其他本机开销等问题。这往往会随着执行程序的大小而增加（通常为6-10％）。 `YARN`和`Kubernetes`目前支持此选项。 | 
| `spark.extraListeners` | (none) | 逗号分隔的实现 `SparkListener` 接口的类名列表；初始化`SparkContext`时，这些类的实例会被创建出来，并且注册到`Spark` 的监听器上。如果这些类有一个接受`SparkConf`作为唯一参数的构造函数，那么这个构造函数会被调用；否则，就调用无参构造函数。如果没有合适的构造函数，`SparkContext` 创建的时候会抛异常。 |
| `spark.local.dir` | /tmp | `Spark` 的"草稿"目录，包括`map`输出的临时文件以及`RDD`存在磁盘上的数据。这个目录最好在本地文件系统中。这个配置可以接受一个以逗号分隔的多个挂载到不同磁盘上的目录列表。注意：`Spark-1.0` 及以后版本中，这个属性会被`cluster manager`设置的环境变量覆盖：`SPARK_LOCAL_DIRS（Standalone，Mesos`）或者 `LOCAL_DIRS（YARN）`。 |
| `spark.logConf` | false | `SparkContext` 启动时是否把生效的 `SparkConf` 属性以 `INFO` 日志打印到日志里。 |
| `spark.master` | (none) | 要连接的 `cluster manager`。参考 [Cluster Manager](https://spark.apache.org/docs/latest/submitting-applications.html#master-urls) 类型。 |
| `spark.submit.deployMode` | (none) | `Spark driver`程序的部署模式，可以是 "client" 或 "cluster"，意味着部署`dirver`程序本地（"client"）或者远程（"cluster"）在 `Spark`集群的其中一个节点上。 |
| `spark.log.callerContext` | (none) | 在`Yarn/HDFS`上运行时将写入`Yarn RM log/HDFS`审核日志的应用程序信息。 它的长度取决于`Hadoop`配置`hadoop.caller.context.max.size`。 它应该简洁，通常最多可包含50个字符。 |
| `spark.driver.supervise` | false | 如果为`true`，则在失败且退出状态为非零时自动重新启动驱动程序。 仅在`Spark`独立模式或`Mesos`集群部署模式下有效。 |

除此之外，还提供以下属性，在某些情况下可能有用：

### 运行环境

| 属性名称 | 默认值 | 含义 |
| --- | --- | --- |
| `spark.driver.extraClassPath` | (none) | 额外的`classpath`条目需预先添加到驱动程序`classpath中`。注意：在客户端模式下，这一套配置不能通过`SparkConf` 直接在应用在应用程序中，因为`JVM`驱动已经启用了。相反，请在配置文件中通过设置`--driver-class-path`选项或者选择默认属性。 |
| `spark.driver.extraJavaOptions` | (none) | 一些额外的`JVM`属性传递给驱动。例如，`GC`设置或其他日志方面设置。注意，设置最大堆大小（-Xmx）是不合法的。最大堆大小设置可以通过在集群模式下设置 `spark.driver.memory` 选项，并且可以通过`--driver-memory` 在客户端模式设置。_注意:_ 在客户端模式下，这一套配置不能通过 `SparkConf` 直接应用在应用程序中，因为`JVM`驱动已经启用了。相反，请在配置文件中通过设置 `--driver-java-options` 选项或者选择默认属性。 |
| `spark.driver.extraLibraryPath` | (none) | 当启动`JVM`驱动程序时设置一个额外的库路径。_注意:_ 在客户端模式下，这一套配置不能通过 `SparkConf` 直接在应用在应用程序中，因为`JVM`驱动已经启用了。相反，请在配置文件中通过设置 `--driver-library-path` 选项或者选择默认属性。 |
| `spark.driver.userClassPathFirst` | false |（实验）在驱动程序加载类库时，用户添加的`Jar`包是否优先于`Spark`自身的`Jar`包。这个特性可以用来缓解冲突引发的依赖性和用户依赖。目前只是实验功能。这是仅用于集群模式。 |
| `spark.executor.extraClassPath` | (none) | 额外的类路径要预先考虑到`executor`的`classpath`。这主要是为与旧版本的`Spark`向后兼容。用户通常不应该需要设置这个选项。 |
| `spark.executor.extraJavaOptions` | (none) | 一些额外的`JVM`属性传递给`executor`。例如，`GC`设置或其他日志方面设置。注意，设置最大堆大小（-Xmx）是不合法的。`Spark`应该使用 `SparkConf`对象或 `Spark` 脚本中使用的 `spark-defaults.conf` 文件中设置。最大堆大小设置可以在 `spark.executor.memory` 进行设置。 |
| `spark.executor.extraLibraryPath` | (none) | 当启动 `JVM` 的可执行程序时设置额外的类库路径。 |
| `spark.executor.logs.rolling.maxRetainedFiles` | (none) | 最新回滚的日志文件将被系统保留。旧的日志文件将被删除。默认情况下禁用。 |
| `spark.executor.logs.rolling.enableCompression` | false | 启用执行程序日志压缩。 如果已启用，则将压缩已滚动的执行程序日志。 默认情况下禁用。 |
| `spark.executor.logs.rolling.maxSize` | (none) | 设置最大文件的大小，以字节为单位日志将被回滚。默认禁用。见 `spark.executor.logs.rolling.maxRetainedFiles` 旧日志的自动清洗。 |
| `spark.executor.logs.rolling.strategy` | (none) | 设置`executor`日志的回滚策略。它可以被设置为 “时间”（基于时间的回滚）或 “大小”（基于大小的回滚）。对于 “时间”，使用 `spark.executor.logs.rolling.time.interval` 设置回滚间隔。用 `spark.executor.logs.rolling.maxSize` 设置最大文件大小回滚。 |
| `spark.executor.logs.rolling.time.interval` | daily | 设定的时间间隔，`executor`日志将回滚。默认情况下是禁用的。有效值是`每天`，`每小时`，`每分钟`或任何时间间隔在几秒钟内。见 `spark.executor.logs.rolling.maxRetainedFiles` 旧日志的自动清洗。 |
| `spark.executor.userClassPathFirst` | false |（实验）与 `spark.driver.userClassPathFirst` 相同的功能，但适用于执行程序的实例。 |
| `spark.executorEnv.[EnvironmentVariableName]` | (none) | 通过添加指定的环境变量 `EnvironmentVariableName` 给 `executor` 进程。用户可以设置多个环境变量。 |
| `spark.redaction.regex` | `(?i)secret|password` | 正则表达式决定驱动程序和执行程序环境中的哪些`Spark`配置属性和环境变量包含敏感信息。 当此正则表达式匹配属性键或值时，将从环境UI和各种日志（如`YARN`和事件日志）中编辑该值。 |
| `spark.python.profile` | false | 启用在 `python` 中的 `profile`。结果将由 `sc.show_profiles()` 显示，或者它将会在驱动程序退出后显示。它还可以通过 `sc.dump_profiles(path)` `dump` 到磁盘。如果一些 `profile` 文件的结果已经显示，那么它们将不会再驱动程序退出后再次显示。默认情况下，`pyspark.profiler.BasicProfiler` 将被使用，但这可以通过传递一个 `profile` 类作为一个参数到 `SparkContext` 中进行覆盖。 |
| `spark.python.profile.dump` | (none) | 这个目录是在驱动程序退出后，`proflie` 文件 `dump` 到磁盘中的文件目录。结果将为每一个 `RDD dump` 为分片文件。它们可以通过 `ptats.Stats()` 加载。如果指定，`profile` 结果将不会自动显示。 |
| `spark.python.worker.memory` | 512m | 在聚合期间，每个`python`工作进程使用的内存量，与`JVM`内存条（例如：`512m`，`2g`）格式相同。如果在聚合过程中使用的内存高于此数量，则会将数据溢出到磁盘中。 |
| `spark.python.worker.reuse` | true | 重用 `python worker`。如果为 `true`，它将使用固定数量的 `worker` 数量。不需要为每一个任务分配 `python `进程。如果是大型的这将是非常有用。 |
| `spark.files` |  | 以逗号分隔的文件列表，放在每个执行程序的工作目录中。 |
| `spark.submit.pyFiles` |  | 以逗号分隔的`.zip`，`.egg`或`.py`文件列表，放在`Python`应用程序的`PYTHONPATH`上。 |
| `spark.jars` |  | 以逗号分隔的本地`jar`列表，包含在驱动程序和执行程序类路径中。 |
| `spark.jars.packages` |  | 以逗号分隔的包含在驱动程序和执行程序类路径上的`jar`的`Maven`坐标列表。 坐标应为`groupId:artifactId:version`。 如果给出`spark.jars.ivySettings`，将根据文件中的配置解析工件，否则将在本地`maven`仓库中搜索工件，然后在`maven central`中搜索，最后由命令行选项提供任何其他远程存储库`--repositories`。 有关详细信息，请参阅[高级依赖管理。](https://spark.apache.org/docs/latest/submitting-applications.html#advanced-dependency-management). |
| `spark.jars.excludes` |  | 逗号分隔的`groupId:artifactId`列表，在解析`spark.jars.packages`中提供的依赖项时排除，以避免依赖性冲突。 |
| `spark.jars.ivy` |  | 指定`Ivy`用户目录的路径，用于来自`spark.jars.packages`的本地`Ivy`缓存和包文件。 这将覆盖常春藤属性`ivy.default.ivy.user.dir`，默认为`〜/.ivy2`。 |
| `spark.jars.ivySettings` |  | 常春藤设置文件的路径，用于自定义使用 `spark.jars.packages`指定的`jar`的分辨率，而不是内置的默认值，例如`maven central`。 还将包括命令行选项`--repositorie`s或`spark.jars.repositories`提供的其他存储库。 用于允许`Spark`从防火墙后面解析工件，例如 通过像`Artifactory`这样的内部工件服务器。 有关设置文件格式的详细信息，请访问http://ant.apache.org/ivy/history/latest-milestone/settings.html |
| `spark.jars.repositories` | | 以逗号分隔的其他远程存储库列表，用于搜索使用`--packages`或`spark.jars.packages`指定的`maven`坐标。 |
| `spark.pyspark.driver.python` |  | `Python`二进制可执行文件，用于驱动程序中的`PySpark`。 （默认为`spark.pyspark.python`） |
| `spark.pyspark.python` |  | `Python`二进制可执行文件，用于驱动程序和执行程序中的`PySpark`。 |

### Shuffle 行为

| 属性名称 | 默认值 | 含义 |
| --- | --- | --- |
| `spark.reducer.maxSizeInFlight` | 48m | 从每个 Reduce 任务中并行的 fetch 数据的最大大小。因为每个输出都要求我们创建一个缓冲区，这代表要为每一个 Reduce 任务分配一个固定大小的内存。除非内存足够大否则尽量设置小一点。 |
| `spark.reducer.maxReqsInFlight` | Int.MaxValue | 在集群节点上，这个配置限制了远程 fetch 数据块的连接数目。当集群中的主机数量的增加时候，这可能导致大量的到一个或多个节点的主动连接，导致负载过多而失败。通过限制获取请求的数量，可以缓解这种情况。 |
| `spark.reducer.maxBlocksInFlightPerAddress` | Int.MaxValue | 此配置限制从给定主机端口每个reduce任务获取的远程块的数量。 当在单次提取或同时从给定地址请求大量块时，这可能使服务执行器或节点管理器崩溃。 在启用外部随机播放时，减少节点管理器上的负载特别有用。 您可以通过将其设置为较低的值来了解此问题。 | 
| `spark.maxRemoteBlockSizeFetchToMem` | Int.MaxValue - 512 | 当块的大小高于此阈值（以字节为单位）时，远程块将被提取到磁盘。 这是为了避免占用过多内存的巨大请求。 默认情况下，仅对大于2GB的块启用此功能，因为无论可用资源是什么，都无法直接将其提取到内存中。 但它可以降低到更低的值（例如200米），以避免在较小的块上使用太多的内存。 请注意，此配置将影响随机提取和块管理器远程块提取。 对于启用了外部随机服务的用户，只有在外部shuffle服务比Spark 2.2更新时才能使用此功能。 | 
| `spark.shuffle.compress` | true | 是否要对 map 输出的文件进行压缩。默认为 true，使用 `spark.io.compression.codec`。 |
| `spark.shuffle.file.buffer` | 32k | 每个 shuffle 文件输出流的内存大小。这些缓冲区的数量减少了磁盘寻道和系统调用创建的 shuffle 文件。 |
| `spark.shuffle.io.maxRetries` | 3 |（仅适用于 Netty）如果设置了非 0 值，与 IO 异常相关失败的 fetch 将自动重试。在遇到长时间的 GC 问题或者瞬态网络连接问题时候，这种重试有助于大量 shuffle 的稳定性。 |
| `spark.shuffle.io.numConnectionsPerPeer` | 1 |（仅Netty）重新使用主机之间的连接，以减少大型集群的连接建立。 对于具有许多硬盘和少量主机的群集，这可能导致并发性不足以使所有磁盘饱和，因此用户可考虑增加此值。 |
| `spark.shuffle.io.preferDirectBufs` | true |（仅适用于 Netty）堆缓冲区用于减少在 shuffle 和缓存块传输中的垃圾回收。对于严格限制的堆内存环境中，用户可能希望把这个设置关闭，以强制Netty的所有分配都在堆上。 |
| `spark.shuffle.io.retryWait` | 5s |（仅适用于 Netty）fetch 重试的等待时长。默认 15s。计算公式是 `maxRetries * retryWait`。 |
| `spark.shuffle.service.enabled` | false | 启用外部随机播放服务。此服务保留由执行者编写的随机播放文件，以便可以安全地删除执行程序。如果`spark.dynamicAllocation.enabled` 为 "true"，则必须启用此功能。必须设置外部随机播放服务才能启用它。有关详细信息，请参阅 [动态分配配置和设置文档](job-scheduling.html#configuration-and-setup)。 |
| `spark.shuffle.service.port` | 7337 | 外部 shuffle 的运行端口。 |
| `spark.shuffle.service.index.cache.size` | 100m | 缓存条目限制为指定的内存占用（以字节为单位） |
| `spark.shuffle.maxChunksBeingTransferred` | Long.MAX_VALUE | 允许在随机服务上同时传输的最大块数。 请注意，当达到最大数量时，将关闭新的传入连接。 客户端将根据`shuffle retry configs`重试（请参阅`spark.shuffle.io.maxRetrie`s和`spark.shuffle.io.retryWait`），如果达到这些限制，任务将失败并且提取失败。 |
| `spark.shuffle.sort.bypassMergeThreshold` | 200 | (Advanced) In the sort-based shuffle manager, avoid merge-sorting data if there is no map-side aggregation and there are at most this many reduce partitions. |
| `spark.shuffle.spill.compress` | true | Whether to compress data spilled during shuffles. Compression will use `spark.io.compression.codec`. |
| `spark.shuffle.accurateBlockThreshold` | 100 * 1024 * 1024 | When we compress the size of shuffle blocks in HighlyCompressedMapStatus, we will record the size accurately if it's above this config. This helps to prevent OOM by avoiding underestimating shuffle block size when fetch shuffle blocks. |
| `spark.shuffle.registration.timeout` | 5000 | 注册到外部shuffle服务的超时（以毫秒为单位）。 | 
| `spark.shuffle.registration.maxAttempts` | 3 | 当我们未能注册到外部shuffle服务时，我们将重试maxAttempts次。 |

### Spark UI

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.eventLog.logBlockUpdates.enabled` | false | 是否为每个块更新记录事件，如果spark.eventLog.enabled为true。 *警告*：这会大大增加事件日志的大小。 |
| `spark.eventLog.longForm.enabled` | false | 如果为true，请在事件日志中使用长形式的调用站点。 否则使用简短形式。 | 
| `spark.eventLog.compress` | false | 是否压缩记录的事件，如果 `spark.eventLog.enabled` 为true。压缩将使用`spark.io.compression.codec`。 |
| `spark.eventLog.dir` | file:///tmp/spark-events | Spark 事件日志的文件路径。如果 `spark.eventLog.enabled` 为 true。在这个基本目录下，Spark 为每个应用程序创建一个二级目录，日志事件特定于应用程序的目录。用户可能希望设置一个统一的文件目录像一个 HDFS 目录那样，所以历史文件可以从历史文件服务器中读取。 |
| `spark.eventLog.enabled` | false | 是否对 Spark 事件记录日志。在应用程序启动后有助于重建 Web UI。 |
| `spark.eventLog.overwrite` | false | 是否覆盖任何现有文件。  |
| `spark.eventLog.buffer.kb` | 100k | 写入输出流时使用的缓冲区大小，以KiB表示，除非另有说明。 |
| `spark.ui.dagGraph.retainedRootRDDs` | Int.MaxValue |  垃圾收集之前，Spark UI和状态API记住了多少个DAG图节点。 |
| `spark.ui.enabled` | true | Whether to run the web UI for the Spark application. |
| `spark.ui.killEnabled` | true | 允许从 Web UI 中结束相应的工作进程。 |
| `spark.ui.liveUpdate.period` | 100ms |更新实时实体的频率。 -1表示重放应用程序时“永不更新”，这意味着只会发生最后一次写入。 对于实时应用程序，这可以避免在快速处理传入任务事件时我们无法进行的一些操作  |
| `spark.ui.liveUpdate.minFlushPeriod` | 1s | 刷新过时的UI数据之前经过的最短时间。 这可以避免在未频繁触发传入任务事件时UI过时。 |
| `spark.ui.port` | 4040 | 应用 UI 的端口，用于显示内存和工作负载数据。 |
| `spark.ui.retainedJobs` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 Job 可以留存。 |
| `spark.ui.retainedStages` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 Stage 可以留存。 |
| `spark.ui.retainedTasks` | 100000 | 在垃圾回收前，Spark UI 和 API 有多少 Task 可以留存。 |
| `spark.ui.reverseProxy` | false | Enable running Spark Master as reverse proxy for worker and application UIs. In this mode, Spark master will reverse proxy the worker and application UIs to enable access without requiring direct access to their hosts. Use it with caution, as worker and application UI will not be accessible directly, you will only be able to access them through spark master/proxy public URL. This setting affects all the workers and application UIs running in the cluster and must be set on all the workers, drivers and masters. |
| `spark.ui.reverseProxyUrl` |  | This is the URL where your proxy is running. This URL is for proxy which is running in front of Spark Master. This is useful when running proxy for authentication e.g. OAuth proxy. Make sure this is a complete URL including scheme (http/https) and port to reach your proxy. |
| `spark.ui.showConsoleProgress` | true | Show the progress bar in the console. The progress bar shows the progress of stages that run for longer than 500ms. If multiple stages run at the same time, multiple progress bars will be displayed on the same line. |
| `spark.worker.ui.retainedExecutors` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 execution 已经完成。 |
| `spark.worker.ui.retainedDrivers` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 driver 已经完成。 |
| `spark.sql.ui.retainedExecutions` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 execution 已经完成。 |
| `spark.streaming.ui.retainedBatches` | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 batch 已经完成。 |
| `spark.ui.retainedDeadExecutors` | 100 | 在垃圾回收前，Spark UI 和 API 有多少 dead executors。 |
| `spark.ui.filters` | None | 逗号分隔的过滤器类名列表，以应用于Spark Web UI。 过滤器应该是标准的javax servlet过滤器。
通过设置表单spark的配置条目，也可以在配置中指定过滤器参数。<过滤器的类名> .param。<param name> = <value>
例如：
`spark.ui.filters= com.test.filter1`
`spark.com.test.filter1.param.name1= FOO`
`spark.com.test.filter1.param.name2=栏` |
| `spark.ui.requestHeaderSize` | 8k | 除非另行指定，否则HTTP请求标头的最大允许大小（以字节为单位）。 此设置也适用于Spark History Server。 |

### Compression and Serialization（压缩和序列化）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.broadcast.compress` | true | 是否在发送之前压缩广播变量。一般是个好主意压缩将使用 `spark.io.compression.codec`。 |
| `spark.io.compression.codec` | lz4 | 内部数据使用的压缩编解码器，如 RDD 分区，广播变量和混洗输出。默认情况下，Spark 提供三种编解码器：`lz4`，`lzf`，和 `snappy`。您还可以使用完全限定类名来指定编码解码器，例如：`org.apache.spark.io.LZ4CompressionCodec`，`org.apache.spark.io.LZFCompressionCodec`，和 `org.apache.spark.io.SnappyCompressionCodec`。 |
| `spark.io.compression.lz4.blockSize` | 32k | 在采用 LZ4 压缩编解码器的情况下，LZ4 压缩使用的块大小。减少块大小还将降低采用 LZ4 时的混洗内存使用。 |
| `spark.io.compression.snappy.blockSize` | 32k | 在采用 Snappy 压缩编解码器的情况下，Snappy 压缩使用的块大小。减少块大小还将降低采用 Snappy 时的混洗内存使用。 |
| `spark.io.compression.zstd.level` | 1 | Zstd压缩编解码器的压缩级别。 增加压缩级别将导致更好的压缩，代价是更多的CPU和内存。|
| `spark.io.compression.zstd.bufferSize` | 32k | 在使用Zstd压缩编解码器的情况下，在Zstd压缩中使用的缓冲区大小（以字节为单位）。 降低此大小将降低使用Zstd时的随机内存使用量，但由于过多的JNI调用开销，可能会增加压缩成本。 | 
| `spark.kryo.classesToRegister` | (none) | 如果你采用 Kryo 序列化，给一个以逗号分隔的自定义类名列以注册 Kryo。有关详细信息，请参阅[调优指南](tuning.html#data-serialization)。 |
| `spark.kryo.referenceTracking` | true | 当使用 Kryo 序列化数据时，是否跟踪对同一对象的引用，如果对象图具有循环，并且如果它们包含同一对象的多个副本对效率有用，则这是必需的。如果您知道这不是这样，可以禁用此功能来提高性能。 |
| `spark.kryo.registrationRequired` | false | 是否需要注册 Kryo。如果设置为 'true'，如果未注册的类被序列化，Kryo 将抛出异常。如果设置为 false（默认值），Kryo 将与每个对象一起写入未注册的类名。编写类名可能会导致显著的性能开销，因此启用此选项可以严格强制用户没有从注册中省略类。 |
| `spark.kryo.registrator` | (none) | 如果你采用 Kryo 序列化，则给一个逗号分隔的类列表，以使用 Kryo 注册你的自定义类。如果你需要以自定义方式注册你的类，则此属性很有用，例如以指定自定义字段序列化程序。否则，使用 spark.kryo.classesToRegisteris 更简单。它应该设置为 [`KryoRegistrator`](api/scala/index.html#org.apache.spark.serializer.KryoRegistrator) 的子类。详见：[调整指南](tuning.html#data-serialization)。 |
| `spark.kryo.unsafe` | false | Whether to use unsafe based Kryo serializer. Can be substantially faster by using Unsafe Based IO. |
| `spark.kryoserializer.buffer.max` | 64m | Kryo 序列化缓冲区的最大允许大小。这必须大于你需要序列化的任何对象。如果你在 Kryo 中得到一个 “buffer limit exceeded” 异常，你就需要增加这个值。 |
| `spark.kryoserializer.buffer` | 64k | Kryo 序列化缓冲区的初始大小。注意，每个 worker上 _每个 core_ 会有一个缓冲区。如果需要，此缓冲区将增长到 `spark.kryoserializer.buffer.max`。 |
| `spark.rdd.compress` | false | 是否压缩序列化RDD分区（例如，在 Java 和 Scala 中为 `StorageLevel.MEMORY_ONLY_SER` 或在 Python 中为 `StorageLevel.MEMORY_ONLY`）。可以节省大量空间，花费一些额外的CPU时间。压缩将使用 `spark.io.compression.codec`。 |
| `spark.serializer` | org.apache.spark.serializer.
JavaSerializer | 用于序列化将通过网络发送或需要以序列化形式缓存的对象的类。Java 序列化的默认值与任何Serializable Java对象一起使用，但速度相当慢，所以我们建议您在需要速度时使用 [使用 `org.apache.spark.serializer.KryoSerializer` 并配置 Kryo 序列化](tuning.html)。可以是 [`org.apache.spark.Serializer`](api/scala/index.html#org.apache.spark.serializer.Serializer) 的任何子类。 |
| `spark.serializer.objectStreamReset` | 100 | 当正使用 org.apache.spark.serializer.JavaSerializer 序列化时，序列化器缓存对象虽然可以防止写入冗余数据，但是却停止这些缓存对象的垃圾回收。通过调用 'reset' 你从序列化程序中清除该信息，并允许收集旧的对象。要禁用此周期性重置，请将其设置为 -1。默认情况下，序列化器会每过 100 个对象被重置一次。 |

### Memory Management（内存管理）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.memory.fraction` | 0.6 | 用于执行和存储的（堆空间 - 300MB）的分数。这个值越低，溢出和缓存数据逐出越频繁。此配置的目的是在稀疏、异常大的记录的情况下为内部元数据，用户数据结构和不精确的大小估计预留内存。推荐使用默认值。有关更多详细信息，包括关于在增加此值时正确调整 JVM 垃圾回收的重要信息，请参阅 [this description](tuning.html#memory-management-overview)。 |
| `spark.memory.storageFraction` | 0.5 | 不会被逐出内存的总量，表示为 `s​park.memory.fraction` 留出的区域大小的一小部分。这个越高，工作内存可能越少，执行和任务可能更频繁地溢出到磁盘。推荐使用默认值。有关更多详细信息，请参阅 [this description](tuning.html#memory-management-overview)。 |
| `spark.memory.offHeap.enabled` | false | 如果为 true，Spark 会尝试对某些操作使用堆外内存。如果启用了堆外内存使用，则 `spark.memory.offHeap.size` 必须为正值。 |
| `spark.memory.offHeap.size` | 0 | 可用于堆外分配的绝对内存量（以字节为单位）。此设置对堆内存使用没有影响，因此如果您的执行器的总内存消耗必须满足一些硬限制，那么请确保相应地缩减JVM堆大小。当 `spark.memory.offHeap.enabled=true` 时，必须将此值设置为正值。 |
| `spark.memory.useLegacyMode` | false | 是否启用 Spark 1.5 及以前版本中使用的传统内存管理模式。传统模式将堆空间严格划分为固定大小的区域，如果未调整应用程序，可能导致过多溢出。必须启用本参数，以下选项才可用：`spark.shuffle.memoryFraction`
`spark.storage.memoryFraction`
`spark.storage.unrollFraction` |
| `spark.shuffle.memoryFraction` | 0.2 |（过时）只有在启用 `spark.memory.useLegacyMode` 时，此属性才是可用的。混洗期间用于聚合和 cogroups 的 Java 堆的分数。在任何给定时间，用于混洗的所有内存映射的集合大小不会超过这个上限，超过该限制的内容将开始溢出到磁盘。如果溢出频繁，请考虑增加此值，但这以 `spark.storage.memoryFraction` 为代价。 |
| `spark.storage.memoryFraction` | 0.6 |（过时）只有在启用 `spark.memory.useLegacyMode` 时，此属性才是可用的。Java 堆的分数，用于 Spark 的内存缓存。这个值不应该大于 JVM 中老生代（old generation) 对象所占用的内存，默认情况下，它提供 0.6 的堆，但是如果配置你所用的老生代对象大小，你可以增加它。 |
| `spark.storage.unrollFraction` | 0.2 |（过时）只有在启用 `spark.memory.useLegacyMode` 时，此属性才是可用的。`spark.storage.memoryFraction` 用于在内存中展开块的分数。当没有足够的空闲存储空间来完全展开新块时，通过删除现有块来动态分配。 |
| `spark.storage.replication.proactive` | false | Enables proactive block replication for RDD blocks. Cached RDD block replicas lost due to executor failures are replenished if there are any existing available replicas. This tries to get the replication level of the block to the initial number. |
| `spark.cleaner.periodicGC.interval` | 30min | 控制触发垃圾回收的频率。仅当弱引用被垃圾收集时，此上下文清除程序才会触发清理。 在具有大型驱动程序JVM的长时间运行的应用程序中，驱动程序上的内存压力很小，这可能偶尔发生或根本不发生。 根本不清理可能会导致执行程序在一段时间后耗尽磁盘空间。|
| `spark.cleaner.referenceTracking` | true | 启用或禁用上下文清理。 | 
| `spark.cleaner.referenceTracking.blocking` | true | 控制清理线程是否应该阻止清除任务（除了shuffle，由`spark.cleaner.referenceTracking.blocking.shuffle Spark`属性控制）。|
| `spark.cleaner.referenceTracking.blocking.shuffle` | false | 控制清理线程是否应阻止随机清理任务。 | 
| `spark.cleaner.referenceTracking.cleanCheckpoints` | false | 控制在引用超出范围时是否清理检查点文件。 |

### Execution Behavior（执行行为）

| Property Name（属性名称）| Default（默认行为）| Meaning（含义）|
| --- | --- | --- |
| `spark.broadcast.blockSize` | 4m | `TorrentBroadcastFactory` 的一个块的每个分片大小。过大的值会降低广播期间的并行性（更慢了）; 但是，如果它过小，`BlockManager` 可能会受到性能影响。|
| `spark.broadcast.checksum` | true | 是否启用广播校验和。 如果启用，广播将包括校验和，这可以帮助检测损坏的块，但代价是计算和发送更多数据。 如果网络具有其他机制来保证数据在广播期间不会被破坏，则可以禁用它。 |
| `spark.executor.cores` | 在 YARN 模式下默认为 1，standlone 和 Mesos 粗粒度模型中的 worker 节点的所有可用的 core。 | 在每个 executor（执行器）上使用的 core 数。在 standlone 和 Mesos 的粗粒度模式下，设置此参数允许应用在相同的 worker 上运行多个 executor（执行器），只要该 worker 上有足够的 core。否则，每个 application（应用）在单个 worker 上只会启动一个 executor（执行器）。 |
| `spark.default.parallelism` | 对于分布式混洗（shuffle）操作，如 `reduceByKey` 和 `join`，父 RDD 中分区的最大数量。对于没有父 RDD 的 `parallelize` 操作，它取决于集群管理器：<br><li>本地模式：本地机器上的 core 数<br><li>Mesos 细粒度模式：8<br><li>其他：所有执行器节点上的 core 总数或者 2，以较大者为准 | 如果用户没有指定参数值，则这个属性是 `join`，`reduceByKey`，和 `parallelize` 等转换返回的 RDD 中的默认分区数。 |
| `spark.executor.heartbeatInterval` | 10s | 每个执行器的心跳与驱动程序之间的间隔。心跳让驱动程序知道执行器仍然存活，并用正在进行的任务的指标更新它 |
| `spark.files.fetchTimeout` | 60s | 获取文件的通讯超时，所获取的文件是从驱动程序通过 SparkContext.addFile() 添加的。 |
| `spark.files.useFetchCache` | true | 如果设置为 true（默认），文件提取将使用由属于同一应用程序的执行器共享的本地缓存，这可以提高在同一主机上运行许多执行器时的任务启动性能。如果设置为 false，这些缓存优化将被禁用，所有执行器将获取它们自己的文件副本。如果使用驻留在 NFS 文件系统上的 Spark 本地目录，可以禁用此优化（有关详细信息，请参阅 [SPARK-6313](https://issues.apache.org/jira/browse/SPARK-6313)）。 |
| `spark.files.overwrite` | false | 当目标文件存在且其内容与源不匹配的情况下，是否覆盖通过 SparkContext.addFile() 添加的文件。 |
| `spark.files.maxPartitionBytes` | 134217728 (128 MB) | The maximum number of bytes to pack into a single partition when reading files. |
| `spark.files.openCostInBytes` | 4194304 (4 MB) | The estimated cost to open a file, measured by the number of bytes could be scanned in the same time. This is used when putting multiple files into a partition. It is better to over estimate, then the partitions with small files will be faster than partitions with bigger files. |
| `spark.hadoop.cloneConf` | false | 如果设置为true，则为每个任务克隆一个新的Hadoop `Configuration` 对象。应该启用此选项以解决 `Configuration` 线程安全问题（有关详细信息，请参阅 [SPARK-2546](https://issues.apache.org/jira/browse/SPARK-2546)）。默认情况下，这是禁用的，以避免不受这些问题影响的作业的意外性能回归。 |
| `spark.hadoop.validateOutputSpecs` | true | 如果设置为 true，则验证 saveAsHadoopFile 和其他变体中使用的输出规范（例如，检查输出目录是否已存在）。可以禁用此选项以静默由于预先存在的输出目录而导致的异常。我们建议用户不要禁用此功能，除非需要实现与以前版本的 Spark 的兼容性。可以简单地使用 Hadoop 的 FileSystem API 手动删除输出目录。对于通过 Spark Streaming 的StreamingContext 生成的作业会忽略此设置，因为在检查点恢复期间可能需要将数据重写到预先存在的输出目录。 |
| `spark.storage.memoryMapThreshold` | 2m | 当从磁盘读取块时，Spark 内存映射的块大小。这会阻止 Spark 从内存映射过小的块。通常，存储器映射对于接近或小于操作系统的页大小的块具有高开销。 |
| `spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version` | 1 | The file output committer algorithm version, valid algorithm version number: 1 or 2. Version 2 may have better performance, but version 1 may handle failures better in certain situations, as per [MAPREDUCE-4815](https://issues.apache.org/jira/browse/MAPREDUCE-4815). |

### Networking（网络）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.rpc.message.maxSize` | 128 | 在 “control plane” 通信中允许的最大消息大小（以 MB 为单位）; 一般只适用于在 executors 和 driver 之间发送的映射输出大小信息。如果您正在运行带有数千个 map 和 reduce 任务的作业，并查看有关 RPC 消息大小的消息，请增加此值。 |
| `spark.blockManager.port` | (random) | 所有块管理器监听的端口。这些都存在于 driver 和 executors 上。 |
| `spark.driver.blockManager.port` | (value of spark.blockManager.port) | Driver-specific port for the block manager to listen on, for cases where it cannot use the same configuration as executors. |
| `spark.driver.bindAddress` | (value of spark.driver.host) | Hostname or IP address where to bind listening sockets. This config overrides the SPARK_LOCAL_IP environment variable (see below).
It also allows a different address from the local one to be advertised to executors or external systems. This is useful, for example, when running containers with bridged networking. For this to properly work, the different ports used by the driver (RPC, block manager and UI) need to be forwarded from the container's host. |
| `spark.driver.host` | (local hostname) | 要监听的 driver 的主机名或 IP 地址。这用于与 executors 和 standalone Master 进行通信。 |
| `spark.driver.port` | (random) | 要监听的 driver 的端口。这用于与 executors 和 standalone Master 进行通信。 |
| `spark.network.timeout` | 120s | 所有网络交互的默认超时。如果未配置此项，将使用此配置替换 `spark.core.connection.ack.wait.timeout`，`spark.storage.blockManagerSlaveTimeoutMs`，`spark.shuffle.io.connectionTimeout`，`spark.rpc.askTimeout` or `spark.rpc.lookupTimeout`。 |
| `spark.port.maxRetries` | 16 | 在绑定端口放弃之前的最大重试次数。当端口被赋予特定值（非 0）时，每次后续重试将在重试之前将先前尝试中使用的端口增加 1。这本质上允许它尝试从指定的开始端口到端口 + maxRetries 的一系列端口。 |
| `spark.rpc.numRetries` | 3 | 在 RPC 任务放弃之前重试的次数。RPC 任务将在此数字的大多数时间运行。 |
| `spark.rpc.retry.wait` | 3s | RPC 请求操作在重试之前等待的持续时间。 |
| `spark.rpc.askTimeout` | `spark.network.timeout` | RPC 请求操作在超时前等待的持续时间。 |
| `spark.rpc.lookupTimeout` | 120s | RPC 远程端点查找操作在超时之前等待的持续时间。 |
| `spark.core.connection.ack.wait.timeout` | spark.network.timeout | 在超时和放弃之前连接等待ack的时间有多长。 为避免因GC等长时间停顿而导致的不必要的超时，您可以设置更大的值。 |

### Scheduling（调度）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.cores.max` | (not set) | 当以 “coarse-grained（粗粒度）” 共享模式在 [standalone deploy cluster](spark-standalone.html) 或 [Mesos cluster in "coarse-grained" sharing mode](running-on-mesos.html#mesos-run-modes) 上运行时，从集群（而不是每台计算机）请求应用程序的最大 CPU 内核数量。如果未设置，默认值将是 Spar k的 standalone deploy 管理器上的 `spark.deploy.defaultCores`，或者 Mesos上的无限（所有可用核心）。 |
| `spark.locality.wait` | 3s | 等待启动本地数据任务多长时间，然后在较少本地节点上放弃并启动它。相同的等待将用于跨越多个地点级别（process-local，node-local，rack-local 等所有）。也可以通过设置 `spark.locality.wait.node` 等来自定义每个级别的等待时间。如果任务很长并且局部性较差，则应该增加此设置，但是默认值通常很好。 |
| `spark.locality.wait.node` | spark.locality.wait | 自定义 node locality 等待时间。例如，您可以将其设置为 0 以跳过 node locality，并立即搜索机架位置（如果群集具有机架信息）。 |
| `spark.locality.wait.process` | spark.locality.wait | 自定义 process locality 等待时间。这会影响尝试访问特定执行程序进程中的缓存数据的任务。 |
| `spark.locality.wait.rack` | spark.locality.wait | 自定义 rack locality 等待时间。 |
| `spark.scheduler.maxRegisteredResourcesWaitingTime` | 30s | 在调度开始之前等待资源注册的最大时间量。 |
| `spark.scheduler.minRegisteredResourcesRatio` | 0.8 for YARN mode; 0.0 for standalone mode and Mesos coarse-grained mode | 注册资源（注册资源/总预期资源）的最小比率（资源是 yarn 模式下的执行程序，standalone 模式下的 CPU 核心和 Mesos coarsed-grained 模式 'spark.cores.max' 值是 Mesos coarse-grained 模式下的总体预期资源]）在调度开始之前等待。指定为 0.0 和 1.0 之间的双精度。无论是否已达到资源的最小比率，在调度开始之前将等待的最大时间量由配置`spark.scheduler.maxRegisteredResourcesWaitingTime` 控制。 |
| `spark.scheduler.mode` | FIFO | 作业之间的 [scheduling mode（调度模式）](job-scheduling.html#scheduling-within-an-application) 提交到同一个 SparkContext。可以设置为 `FAIR` 使用公平共享，而不是一个接一个排队作业。对多用户服务有用。 |
| `spark.scheduler.revive.interval` | 1s | 调度程序复活工作资源去运行任务的间隔长度。 |
| `spark.scheduler.listenerbus.eventqueue.capacity` | 10000 | Spark侦听器总线中事件队列的容量必须大于0.如果删除侦听器事件，请考虑增加值（例如20000）。 增加此值可能会导致驱动程序使用更多内存。 | 
| `spark.scheduler.blacklist.unschedulableTaskSetTimeout` | 120s | 在中止由于完全列入黑名单而无法调度的TaskSet之前等待获取新执行程序并安排任务的超时秒数。 |
| `spark.blacklist.enabled` | false | If set to "true", prevent Spark from scheduling tasks on executors that have been blacklisted due to too many task failures. The blacklisting algorithm can be further controlled by the other "spark.blacklist" configuration options. |
| `spark.blacklist.timeout` | 1h | (Experimental) How long a node or executor is blacklisted for the entire application, before it is unconditionally removed from the blacklist to attempt running new tasks. |
| `spark.blacklist.task.maxTaskAttemptsPerExecutor` | 1 | (Experimental) For a given task, how many times it can be retried on one executor before the executor is blacklisted for that task. |
| `spark.blacklist.task.maxTaskAttemptsPerNode` | 2 | (Experimental) For a given task, how many times it can be retried on one node, before the entire node is blacklisted for that task. |
| `spark.blacklist.stage.maxFailedTasksPerExecutor` | 2 | (Experimental) How many different tasks must fail on one executor, within one stage, before the executor is blacklisted for that stage. |
| `spark.blacklist.stage.maxFailedExecutorsPerNode` | 2 | (Experimental) How many different executors are marked as blacklisted for a given stage, before the entire node is marked as failed for the stage. |
| `spark.blacklist.application.maxFailedTasksPerExecutor` | 2 | (Experimental) How many different tasks must fail on one executor, in successful task sets, before the executor is blacklisted for the entire application. Blacklisted executors will be automatically added back to the pool of available resources after the timeout specified by `spark.blacklist.timeout`. Note that with dynamic allocation, though, the executors may get marked as idle and be reclaimed by the cluster manager. |
| `spark.blacklist.application.maxFailedExecutorsPerNode` | 2 | (Experimental) How many different executors must be blacklisted for the entire application, before the node is blacklisted for the entire application. Blacklisted nodes will be automatically added back to the pool of available resources after the timeout specified by `spark.blacklist.timeout`. Note that with dynamic allocation, though, the executors on the node may get marked as idle and be reclaimed by the cluster manager. |
| `spark.blacklist.application.maxFailedTasksPerExecutor` | 2 | （实验）在执行程序被列入整个应用程序的黑名单之前，在一个执行程序中，在成功的任务集中，有多少个不同的任务必须失败。 在spark.blacklist.timeout指定的超时后，列入黑名单的执行程序将自动添加回可用资源池。 请注意，通过动态分配，执行程序可能会被标记为空闲并由集群管理器回收。 |
| `spark.blacklist.application.maxFailedExecutorsPerNode` | 2 | （实验）在将节点列入整个应用程序的黑名单之前，必须将多个不同的执行程序列入黑名单。 在spark.blacklist.timeout指定的超时后，列入黑名单的节点将自动添加回可用资源池。 但请注意，通过动态分配，节点上的执行程序可能会被标记为空闲并由集群管理器回收。 |
| `spark.blacklist.killBlacklistedExecutors` | false | (Experimental) If set to "true", allow Spark to automatically kill, and attempt to re-create, executors when they are blacklisted. Note that, when an entire node is added to the blacklist, all of the executors on that node will be killed. |
| `spark.blacklist.application.fetchFailure.enabled` | false | （实验）如果设置为“true”，Spark会在发生提取失败时立即将执行程序列入黑名单。 如果启用了外部随机服务，则整个节点将被列入黑名单。 |
| `spark.speculation` | false | 如果设置为 "true"，则执行任务的推测执行。这意味着如果一个或多个任务在一个阶段中运行缓慢，则将重新启动它们。 |
| `spark.speculation.interval` | 100ms | Spark 检查要推测的任务的时间间隔。 |
| `spark.speculation.multiplier` | 1.5 | 一个任务的速度可以比推测的平均值慢多少倍。 |
| `spark.speculation.quantile` | 0.75 | 对特定阶段启用推测之前必须完成的任务的分数。 |
| `spark.task.cpus` | 1 | 要为每个任务分配的核心数。 |
| `spark.task.maxFailures` | 4 | 放弃作业之前任何特定任务的失败次数。分散在不同任务中的故障总数不会导致作业失败; 一个特定的任务允许失败这个次数。应大于或等于 1\. 允许重试次数=此值 - 1\. |
| `spark.task.reaper.enabled` | false | Enables monitoring of killed / interrupted tasks. When set to true, any task which is killed will be monitored by the executor until that task actually finishes executing. See the other `spark.task.reaper.*` configurations for details on how to control the exact behavior of this monitoring. When set to false (the default), task killing will use an older code path which lacks such monitoring. |
| `spark.task.reaper.pollingInterval` | 10s | When `spark.task.reaper.enabled = true`, this setting controls the frequency at which executors will poll the status of killed tasks. If a killed task is still running when polled then a warning will be logged and, by default, a thread-dump of the task will be logged (this thread dump can be disabled via the `spark.task.reaper.threadDump` setting, which is documented below). |
| `spark.task.reaper.threadDump` | true | When `spark.task.reaper.enabled = true`, this setting controls whether task thread dumps are logged during periodic polling of killed tasks. Set this to false to disable collection of thread dumps. |
| `spark.task.reaper.killTimeout` | -1 | When `spark.task.reaper.enabled = true`, this setting specifies a timeout after which the executor JVM will kill itself if a killed task has not stopped running. The default value, -1, disables this mechanism and prevents the executor from self-destructing. The purpose of this setting is to act as a safety-net to prevent runaway uncancellable tasks from rendering an executor unusable. |
| `spark.stage.maxConsecutiveAttempts` | 4 | Number of consecutive stage attempts allowed before a stage is aborted. |

### Dynamic Allocation（动态分配）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.dynamicAllocation.enabled` | false | 是否使用动态资源分配，它根据工作负载调整为此应用程序注册的执行程序数量。有关更多详细信息，请参阅 [here](job-scheduling.html#dynamic-resource-allocation) 的说明。这需要设置 `spark.shuffle.service.enabled`。以下配置也相关：`spark.dynamicAllocation.minExecutors`，`spark.dynamicAllocation.maxExecutors` 和`spark.dynamicAllocation.initialExecutors`。 |
| `spark.dynamicAllocation.executorIdleTimeout` | 60s | 如果启用动态分配，并且执行程序已空闲超过此持续时间，则将删除执行程序。有关更多详细信息，请参阅此[description](job-scheduling.html#resource-allocation-policy)。 |
| `spark.dynamicAllocation.cachedExecutorIdleTimeout` | infinity | 如果启用动态分配，并且已缓存数据块的执行程序已空闲超过此持续时间，则将删除执行程序。有关详细信息，请参阅此 [description](job-scheduling.html#resource-allocation-policy)。 |
| `spark.dynamicAllocation.initialExecutors` | `spark.dynamicAllocation.minExecutors` | 启用动态分配时要运行的执行程序的初始数。如果 `--num-executors`（或 `spark.executor.instances`）被设置并大于此值，它将被用作初始执行器数。 |
| `spark.dynamicAllocation.maxExecutors` | infinity | 启用动态分配的执行程序数量的上限。 |
| `spark.dynamicAllocation.minExecutors` | 0 | 启用动态分配的执行程序数量的下限。 |
| `spark.dynamicAllocation.executorAllocationRatio` | 1 | 默认情况下，动态分配将请求足够的执行程序，以根据要处理的任务数最大化并行度。 虽然这可以最大限度地减少作业的延迟，但由于执行程序的分配开销，这个设置可能会浪费大量资源，因为某些执行程序可能甚至不做任何工作。 此设置允许设置将用于减少w.r.t执行程序数量的比率。 完全并行。 默认为1.0以提供最大并行度。 0.5将执行程序的目标数除以2由dynamicAllocation计算的执行程序的目标数仍然可以被`spark.dynamicAllocation.minExecutors和spark.dynamicAllocation.maxExecutors`设置覆盖 |
| `spark.dynamicAllocation.schedulerBacklogTimeout` | 1s | 如果启用动态分配，并且有超过此持续时间的挂起任务积压，则将请求新的执行者。有关更多详细信息，请参阅此 [description](job-scheduling.html#resource-allocation-policy)。 |
| `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` | `schedulerBacklogTimeout` | 与 `spark.dynamicAllocation.schedulerBacklogTimeout` 相同，但仅用于后续执行者请求。有关更多详细信息，请参阅此 [description](job-scheduling.html#resource-allocation-policy)。 |

### Security（安全）

有关如何保护不同Spark子系统的可用选项，请参阅[安全性](https://spark.apache.org/docs/latest/security.html)页面。

### Spark SQL

运行 `SET -v` 命令将显示 SQL 配置的整个列表.

```
// spark is an existing SparkSession
spark.sql("SET -v").show(numRows = 200, truncate = false)
```

```
// spark is an existing SparkSession
spark.sql("SET -v").show(200, false);
```

```
# spark is an existing SparkSession
spark.sql("SET -v").show(n=200, truncate=False)
```

```
sparkR.session()
properties <- sql("SET -v")
showDF(properties, numRows = 200, truncate = FALSE)
```

### Spark Streaming

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.streaming.backpressure.enabled` | false | 开启或关闭 Spark Streaming 内部的 backpressure mecheanism（自 1.5 开始）。基于当前批次调度延迟和处理时间，这使得 Spark Streaming 能够控制数据的接收率，因此，系统接收数据的速度会和系统处理的速度一样快。从内部来说，这动态地设置了 receivers 的最大接收率。这个速率上限通过 `spark.streaming.receiver.maxRate` 和 `spark.streaming.kafka.maxRatePerPartition` 两个参数设定（如下）。 |
| `spark.streaming.backpressure.initialRate` | not set | 当 backpressure mecheanism 开启时，每个 receiver 接受数据的初始最大值。 |
| `spark.streaming.blockInterval` | 200ms | 在这个时间间隔（ms）内，通过 Spark Streaming receivers 接收的数据在保存到 Spark 之前，chunk 为数据块。推荐的最小值为 50ms。具体细节见 Spark Streaming 指南的 [performance tuning](streaming-programming-guide.html#level-of-parallelism-in-data-receiving) 一节。 |
| `spark.streaming.receiver.maxRate` | not set | 每秒钟每个 receiver 将接收的数据的最大速率（每秒钟的记录数目）。有效的情况下，每个流每秒将最多消耗这个数目的记录。设置这个配置为 0 或者 -1 将会不作限制。细节参见 Spark Streaming 编程指南的 [deployment guide](streaming-programming-guide.html#deploying-applications) 一节。 |
| `spark.streaming.receiver.writeAheadLog.enable` | false | 为 receiver 启用 write ahead logs。所有通过接收器接收输入的数据将被保存到 write ahead logs，以便它在驱动程序故障后进行恢复。见星火流编程指南部署指南了解更多详情。细节参见 Spark Streaming 编程指南的 [deployment guide](streaming-programming-guide.html#deploying-applications) 一节。 |
| `spark.streaming.unpersist` | true | 强制通过 Spark Streaming 生成并持久化的 RDD 自动从 Spark 内存中非持久化。通过 Spark Streaming 接收的原始输入数据也将清除。设置这个属性为 false 允许流应用程序访问原始数据和持久化 RDD，因为它们没有被自动清除。但是它会造成更高的内存花费。 |
| `spark.streaming.stopGracefullyOnShutdown` | false | 如果为 `true`，Spark 将 gracefully（缓慢地）关闭在 JVM 运行的 StreamingContext，而非立即执行。 |
| `spark.streaming.kafka.maxRatePerPartition` | not set | 在使用新的 Kafka direct stream API 时，从每个 kafka 分区读到的最大速率（每秒的记录数目）。详见 [Kafka Integration guide](streaming-kafka-integration.html)。 |
| `spark.streaming.kafka.maxRetries` | 1 | driver 连续重试的最大次数，以此找到每个分区 leader 的最近的（latest）的偏移量（默认为 1 意味着 driver 将尝试最多两次）。仅应用于新的 kafka direct stream API。 |
| `spark.streaming.ui.retainedBatches` | 1000 | 在垃圾回收之前，Spark Streaming UI 和状态API 所能记得的 批处理（batches）数量。 |
| `spark.streaming.driver.writeAheadLog.closeFileAfterWrite` | false | 在写入一条 driver 中的 write ahead log 记录 之后，是否关闭文件。如果你想为 driver 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing），设定为 true。
| `spark.streaming.receiver.writeAheadLog.closeFileAfterWrite` | false | 在写入一条 reveivers 中的 write ahead log 记录 之后，是否关闭文件。如果你想为 reveivers 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing），设定为 true。 |

### SparkR

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.r.numRBackendThreads` | 2 | 使用 RBackend 处理来自 SparkR 包中的 RPC 调用的线程数。 |
| `spark.r.command` | Rscript | 在 driver 和 worker 两种集群模式下可执行的 R 脚本。 |
| `spark.r.driver.command` | spark.r.command | 在 driver 的 client 模式下可执行的 R 脚本。在集群模式下被忽略。 |
| `spark.r.shell.command` | R | Executable for executing sparkR shell in client modes for driver. Ignored in cluster modes. It is the same as environment variable `SPARKR_DRIVER_R`, but take precedence over it. `spark.r.shell.command` is used for sparkR shell while `spark.r.driver.command` is used for running R script. |
| `spark.r.backendConnectionTimeout` | 6000 | Connection timeout set by R process on its connection to RBackend in seconds. |
| `spark.r.heartBeatInterval` | 100 | Interval for heartbeats sent from SparkR backend to R process to prevent connection timeout. |

### GraphX

| Property Name | Default | Meaning |
| --- | --- | --- |
| `spark.graphx.pregel.checkpointInterval` | -1 | Checkpoint interval for graph and message in Pregel. It used to avoid stackOverflowError due to long lineage chains after lots of iterations. The checkpoint is disabled by default. |

### Deploy（部署）

| Property Name（属性名称）| Default（默认值）| Meaning（含义）|
| --- | --- | --- |
| `spark.deploy.recoveryMode` | NONE | 集群模式下，Spark jobs 执行失败或者重启时，恢复提交 Spark jobs 的恢复模式设定。 |
| `spark.deploy.zookeeper.url` | None | 当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER，这一配置被用来连接 zookeeper URL。 |
| `spark.deploy.zookeeper.dir` | None | 当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER，这一配置被用来设定 zookeeper 目录为 store recovery state。 |

### Cluster Managers（集群管理器）

Spark 中的每个集群管理器都有额外的配置选项，这些配置可以在每个模式的页面中找到:

#### [YARN](running-on-yarn.html#configuration)

#### [Mesos](running-on-mesos.html#configuration)

#### [Standalone Mode](spark-standalone.html#cluster-launch-scripts)

# 环境变量

通过环境变量配置特定的 Spark 设置。环境变量从 Spark 安装目录下的 `conf/spark-env.sh` 脚本读取（或者是 window 环境下的 `conf/spark-env.cmd`）。在 Standalone 和 Mesos 模式下，这个文件可以指定机器的特定信息，比如 hostnames。它也可以为正在运行的 Spark Application 或者提交脚本提供 sourced（来源）.
注意，当 Spark 被安装，默认情况下 `conf/spark-env.sh` 是不存在的。但是，你可以通过拷贝 `conf/spark-env.sh.template` 来创建它。确保你的拷贝文件时可执行的。`spark-env.sh`：中有有以下变量可以被设置 :

| Environment Variable（环境变量）| Meaning（含义）|
| --- | --- |
| `JAVA_HOME` | Java 的安装路径（如果不在你的默认 `PATH` 下）。 |
| `PYSPARK_PYTHON` | 在 driver 和 worker 中 PySpark 用到的 Python 二进制可执行文件（如何有默认为 `python2.7`，否则为 `python`）。如果设置了属性 `spark.pyspark.python`，则会优先考虑。 |
| `PYSPARK_DRIVER_PYTHON` | 只在 driver 中 PySpark 用到的 Python 二进制可执行文件（默认为 `PYSPARK_PYTHON`）。如果设置了属性 `spark.pyspark.driver.python` ,则优先考虑。 |
| `SPARKR_DRIVER_R` | SparkR shell 用到的 R 二进制可执行文件（默认为 `R`）。如果设置了属性 `spark.r.shell.command` 则会优先考虑。 |
| `SPARK_LOCAL_IP` | 机器绑定的 IP 地址。 |
| `SPARK_PUBLIC_DNS` | 你的 Spark 程序通知其他机器的 Hostname。 |

除了以上参数，[standalone cluster scripts](spark-standalone.html#cluster-launch-scripts) 也可以设置其他选项，比如每个机器使用的 CPU 核数和最大内存.

因为 `spark-env.sh` 是 shell 脚本，一些可以通过程序的方式来设置，比如你可以通过特定的网络接口来计算 `SPARK_LOCAL_IP`。

注意：当以 `cluster` mode（集群模式）运行 Spark on YARN 时，环境变量需要通过在您的 `conf/spark-defaults.conf` 文件中 `spark.yarn.appMasterEnv.[EnvironmentVariableName]` 来设定。`cluster` mode（集群模式）下，`spark-env.sh` 中设定的环境变量将不会在 YARN Application Master 过程中反应出来。详见 [YARN-related Spark Properties](running-on-yarn.html#spark-properties).

# 配置 Logging

Spark 用 [log4j](http://logging.apache.org/log4j/) 生成日志，你可以通过在 `conf` 目录下添加 `log4j.properties` 文件来配置.一种方法是拷贝 `log4j.properties.template` 文件.

# Overriding configuration directory（覆盖配置目录）

如果你想指定不同的配置目录，而不是默认的 “SPARK_HOME/conf”，你可以设置 SPARK_CONF_DIR。Spark 将从这一目录下读取文件（spark-defaults.conf，spark-env.sh，log4j.properties 等）

# Inheriting Hadoop Cluster Configuration（继承 Hadoop 集群配置）

如果你想用 Spark 来读写 HDFS，在 Spark 的 classpath 就需要包括两个 Hadoop 配置文件:

*   `hdfs-site.xml`，为 HDFS client 提供 default behaviors（默认的行为）.
*   `core-site.xml`，设定默认的文件系统名称.

这些配置文件的位置因 Hadoop 版本而异，但是一个常见的位置在 `/etc/hadoop/conf` 内。一些工具创建配置 on-the-fly，但提供了一种机制来下载它们的副本.

为了使这些文件对 Spark 可见，需要设定 `$SPARK_HOME/spark-env.sh` 中的 `HADOOP_CONF_DIR` 到一个包含配置文件的位置.

# 自定义Hadoop/Hive配置
如果您的Spark应用程序正在与Hadoop，Hive或两者进行交互，则Spark的类路径中可能存在Hadoop / Hive配置文件。

多个正在运行的应用程序可能需要不同的Hadoop / Hive客户端配置。 您可以在Spark的类路径中为每个应用程序复制和修改`hdfs-site.xml`，`core-site.xml`，`yarn-site.xml`，`hive-site.xml`。 在YARN上运行的Spark群集中，这些配置文件在群集范围内设置，并且无法由应用程序安全地更改。

更好的选择是以spark.hadoop。*的形式使用spark hadoop属性。 它们可以被认为与普通火花属性相同，可以在`$SPARK_HOME/conf spark-defaults.conf`中设置

在某些情况下，您可能希望避免在SparkConf中对某些配置进行硬编码。 例如，Spark允许您简单地创建一个空conf并设置`spark/spark` hadoop属性。
```
val conf = new SparkConf().set("spark.hadoop.abc.def","xyz")
val sc = new SparkContext(conf)
```
此外，您可以在运行时修改或添加配置：

```
./bin/spark-submit \ 
  --name "My app" \ 
  --master local[4] \  
  --conf spark.eventLog.enabled=false \ 
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" \ 
  --conf spark.hadoop.abc.def=xyz \ 
  myApp.jar
```