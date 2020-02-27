# Monitoring and Instrumentation

有几种方法来监视`Spark`应用程序：`Web UI`，`metrics` 和外部工具。

# Web 界面

每个 `SparkContext` 都会启动一个 `Web UI`，默认端口为`4040`，显示有关应用程序的有用信息。这包括：

*   调度器阶段和任务的列表
*   `RDD` 大小和内存使用的概要信息
*   环境信息
*   正在运行的执行器的信息

您可以通过在 `Web` 浏览器中打开 `http://<driver-node>:4040` 来访问此界面。如果多个 `SparkContexts` 在同一主机上运行，则它们将绑定到连续的端口从`4040（4041，4042 etc）`开始。

请注意，默认情况下此信息仅适用于运行中的应用程序。要在事后还能通过 `Web UI` 查看，请在应用程序启动之前，将 `spark.eventLog.enabled` 设置为 `true`。这配置 `Spark` 持久存储以记录 `Spark` 事件，再通过编码该信息在`UI`中进行显示。

## 事后查看

仍然可以通过 `Spark` 的历史服务器构建应用程序的`UI`，只要应用程序的事件日志存在。您可以通过执行以下命令启动历史服务器：

```
./sbin/start-history-server.sh 
```

默认情况下，会在 `http://<server-url>:18080` 创建一个 `Web` 界面，显示未完成、完成以及其他尝试的任务信息。

当使用 `file-system` 提供程序类（见下面 `spark.history.provider`）时，基本日志记录目录必须在`spark.history.fs.logDirectory`配置选项中提供，并且应包含每个代表应用程序的事件日志的子目录。

`Spark` 任务本身必须配置启用记录事件，并将其记录到相同共享的可写目录下。例如，如果服务器配置了日志目录`hdfs://namenode/shared/spark-logs`，那么客户端选项将是：

```
spark.eventLog.enabled true
spark.eventLog.dir hdfs://namenode/shared/spark-logs 
```

history server 可以配置如下：

### 环境变量

| 环境变量 | 含义 |
| --- | --- |
| `SPARK_DAEMON_MEMORY` | `history server` 内存分配（默认值：1g）|
| `SPARK_DAEMON_JAVA_OPTS` | `history server JVM` 选项（默认值：无）|
| `SPARK_DAEMON_CLASSPATH` | 历史服务器的类路径（默认值：无）。 | 
| `SPARK_PUBLIC_DNS` | `history server` 公共地址。如果没有设置，应用程序历史记录的链接可能会使用服务器的内部地址，导致链接断开（默认值：无）。 |
| `SPARK_HISTORY_OPTS` | `spark.history.*` `history server` 配置选项（默认值：无）|

### Spark历史服务配置选项

[安全性](https://spark.apache.org/docs/latest/security.html#web-ui)页面中详细介绍了Spark History Server的安全选项。

| 属性名称 | 默认 | 含义 |
| --- | --- | --- |
| `spark.history.provider` | `org.apache.spark.deploy.history.FsHistoryProvider` | 执行应用程序历史后端的类的名称。目前只有一个实现，由 `Spark` 提供，它查找存储在文件系统中的应用程序日志。 |
| `spark.history.fs.logDirectory` | `file:/tmp/spark-events` | 为了文件系统的历史提供者，包含要加载的应用程序事件日志的目录`UR`L。这可以是 local `file://` 路径，`HDFS` `hdfs://namenode/shared/spark-logs` 或者是 `Hadoop API` 支持的替代文件系统。 |
| `spark.history.fs.update.interval` | 10s | 文件系统历史的提供者在日志目录中检查新的或更新的日志期间。更短的时间间隔可以更快地检测新的应用程序，而不必更多服务器负载重新读取更新的应用程序。一旦更新完成，完成和未完成的应用程序的列表将反映更改。 |
| `spark.history.retainedApplications` | 50 | 在缓存中保留 `UI` 数据的应用程序数量。如果超出此上限，则最早的应用程序将从缓存中删除。如果应用程序不在缓存中，如果从`UI` 界面访问它将不得不从磁盘加载。 |
| `spark.history.ui.maxApplications` | `Int.MaxValue` | 在历史记录摘要页面上显示的应用程序数量。应用程序 `UI`仍然可以通过直接访问其 `URL`，即使它们不显示在历史记录摘要页面上。 |
| `spark.history.ui.port` | 18080 | `history server` 的`Web`界面绑定的端口。 |
| `spark.history.kerberos.enabled` | `false` | 表明 `history server` 是否应该使用 `kerberos` 进行登录。如果 `history server` 正在访问安全的 `Hadoop` 集群上的 HDFS 文件，则需要这样做。如果这是真的，它使用配置 `spark.history.kerberos.principal` 和 `spark.history.kerberos.keytab` |
| `spark.history.kerberos.principal` | (none) | `history server` 的 `Kerberos` 主要名称。 |
| `spark.history.kerberos.keytab` | (none) | `history server` 的 `kerberos keytab` 文件的位置。 |
| `spark.history.fs.cleaner.enabled` | `false` | 指定 `History Server` 是否应该定期从存储中清除事件日志。 |
| `spark.history.fs.cleaner.interval` | 1d | 文件系统 `job history` 清洁程序多久检查要删除的文件。如果文件比 `spark.history.fs.cleaner.maxAge` 更旧，那么它们将被删除。 |
| `spark.history.fs.cleaner.maxAge` | 7d | 较早的 `Job history` 文件将在文件系统历史清除程序运行时被删除。 |
| `spark.history.fs.endEventReparseChunkSize` | 1m | 查找结束事件的日志文件末尾要解析的字节数。 这用于通过跳过事件日志文件的不必要部分来加速应用程序列表的生成。 可以通过将此配置设置为0来禁用它。|
| `spark.history.fs.inProgressOptimization.enabled`	 | `true` | 启用对正在进行的日志的优化处理。 此选项可能会使完成的应用程序无法重命名列为正在进行的事件日志。|
| `spark.history.fs.numReplayThreads` | 25% of available cores | `history server` 用于处理事件日志的线程数。 |
| `spark.history.store.maxDiskUsage`| 10g | 存储缓存应用程序历史记录信息的本地目录的最大磁盘使用量。|
| `spark.history.store.path` | (none) | 本地目录缓存应用程序历史数据的位置。 如果设置，则历史服务器将应用程序数据存储在磁盘上，而不是将其保留在内存中。 写入磁盘的数据将在历史服务器重新启动时重新使用。|

请注意UI中所有的任务，表格可以通过点击它们的标题来排序，便于识别慢速任务，数据偏移等。

注意

1.  `history server` 显示完成的和未完成的` Spark` 作业。如果应用程序在失败后进行多次尝试，将显示失败的尝试，以及任何持续未完成的尝试或最终成功的尝试。

2.  未完成的程序只会间歇性地更新。更新的时间间隔由更改文件的检查间隔（`spark.history.fs.update.interval`）定义。在较大的集群上，更新间隔可能设置为较大的值。查看正在运行的应用程序的方式实际上是查看自己的`Web UI`。

3.  没有注册完成就退出的应用程序将被列出为未完成的，即使它们不再运行。如果应用程序崩溃，可能会发生这种情况。

4.  一个用于表示完成 `Spark` 作业的一种方法是明确地停止`Spark Context`（`sc.stop()`），或者在 `Pytho`n 中使用 `with SparkContext() as sc:` 构造处理 `Spark` 上下文设置并拆除。

## REST API

除了在`UI`中查看指标之外，还可以使用`JSON`。这为开发人员提供了一种简单的方法来为 `Spark` 创建新的可视化和监控工具。`JSON`可用于运行的应用程序和 `history server`。 端点挂载在`/api/v1`。例如，对于 `history server`，它们通常可以在 `http://<server-url>:18080/api/v1` 访问，对于正在运行的应用程序，在 `http://localhost:4040/api/v1`。

在 `API` 中，一个应用程序被其应用程序 `ID` `[app-id]` 引用。当运行在 `YARN`上时，每个应用程序可能会有多次尝试，但是仅针对群集模式下的应用程序进行尝试，而不是客户端模式下的应用程序。`YARN` 群集模式中的应用程序可以通过它们的 `[attempt-id]` 来识别。在下面列出的 `API` 中，当以 YARN集群模式运行时，`[app-id]` 实际上是 `[base-app-id]/[attempt-id]`，其中 `[base-app-id]` `YARN` 应用程序 `ID`。

| Endpoint | 含义 |
| --- | --- |
| `/applications` | 所有应用程序的列表。<br>`?status=[completed&#124;running]` 列出所选状态下的应用程序。<br>`?minDate=[date]` 列出最早的开始日期/时间。<br>`?maxDate=[date]` 列出最新开始日期/时间。<br>`?minEndDate=[date]` 列出最早的结束日期/时间。<br>`?maxEndDate=[date]` 列出最新结束日期/时间。<br>`?limit=[limit]` 限制列出的应用程序数量。<br>示例:<br>`?minDate=2015-02-10`<br>`?minDate=2015-02-03T16:42:40.000GMT`<br>`?maxDate=2015-02-11T20:41:30.000GMT`<br>`?minEndDate=2015-02-12`<br>`?minEndDate=2015-02-12T09:15:10.000GMT`<br>`?maxEndDate=2015-02-14T16:30:45.000GMT`<br>`?limit=10` |
| `/applications/[app-id]/jobs` | 给定应用程序的所有 `job` 的列表。<br>`?status=[running&#124;succeeded&#124;failed&#124;unknown]` 列出在特定状态下的 job。 |
| `/applications/[app-id]/jobs/[job-id]` | 给定 `job` 的详细信息。 |
| `/applications/[app-id]/stages` | 给定应用程序的所有阶段的列表。<br>`?status=[active&#124;complete&#124;pending&#124;failed]` 仅列出状态的阶段。 |
| `/applications/[app-id]/stages/[stage-id]` | 给定阶段的所有尝试的列表。 |
| `/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]` | 给定阶段的尝试详细信息。 |
| `/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskSummary` | 给定阶段尝试中所有任务的汇总指标。<br>`?quantiles` 用给定的分位数总结指标。
Example: `?quantiles=0.01,0.5,0.99` |
| `/applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskList` | 给定阶段尝试的所有 `task` 的列表。<br>`?offset=[offset]&length=[len]` 列出给定范围内的 task。<br>`?sortBy=[runtime&#124;-runtime]` `task` 排序.<br>Example: `?offset=10&length=50&sortBy=runtime` |
| `/applications/[app-id]/executors` | 给定应用程序的所有活动 `executor` 的列表。 |
| `/applications/[app-id]/executors/[executor-id]/threads` | 堆栈在给定活动执行程序中运行的所有线程的跟踪。 通过历史服务器不可用。 | 
| `/applications/[app-id]/allexecutors` | 给定应用程序的所有（活动和死亡）`executor` 的列表。 |
| `/applications/[app-id]/storage/rdd` | 给定应用程序的存储 `RDD` 列表。 |
| `/applications/[app-id]/storage/rdd/[rdd-id]` | 给定 `RDD` 的存储状态的详细信息。 |
| `/applications/[base-app-id]/logs` | 将给定应用程序的所有尝试的事件日志下载为一个 `zip` 文件。 |
| `/applications/[base-app-id]/[attempt-id]/logs` | 将特定应用程序尝试的事件日志下载为一个 `zip` 文件。 |
| `/applications/[app-id]/streaming/statistics` | `streaming context` 的统计信息 |
| `/applications/[app-id]/streaming/receivers` | 所有 `streaming receivers` 的列表。 |
| `/applications/[app-id]/streaming/receivers/[stream-id]` | 给定 `receiver` 的详细信息。 |
| `/applications/[app-id]/streaming/batches` | 所有被保留 `batch` 的列表。 |
| `/applications/[app-id]/streaming/batches/[batch-id]` | 给定 `batch` 的详细信息。 |
| `/applications/[app-id]/streaming/batches/[batch-id]/operations` | 给定 `batch` 的所有输出操作的列表。 |
| `/applications/[app-id]/streaming/batches/[batch-id]/operations/[outputOp-id]` | 给定操作和给定 `batch` 的详细信息。 |
| `/applications/[app-id]/environment` | 给定应用程序环境的详细信息。 |
| `/version` | 获取当前的`spark`版本。 | 

可检索的 `job` 和 `stage` 的数量被 `standalone Spark UI` 的相同保留机制所约束。`"spark.ui.retainedJobs"` 定义触发 `job` 垃圾收集的阈值，以及`spark.ui.retainedStages` 限定 `stage`。请注意，垃圾回收在 `play` 时进行：可以通过增加这些值并重新启动 `history server` 来检索更多条目。

### 执行者任务指标
`REST API`以任务执行的粒度公开Spark执行器收集的任务度量标准的值。 度量标准可用于性能故障排除和工作负载表征。 可用指标的列表，简短说明：

| `spark`执行者任务指标 | 简短描述 |
| --- | --- |
| `executorRunTime` | 执行程序花费的时间来执行此任务。 这包括获取随机数据的时间。 该值以毫秒表示。 |
| `executorCpuTime` | 执行程序花在运行此任务上的CPU时间。 这包括获取随机数据的时间。 该值以纳秒表示。 |
| `executorDeserializeTime` | 花费在反序列化此任务上的时间。 该值以毫秒表示。 |
| `executorDeserializeCpuTime` | 执行程序为反序列化此任务所花费的CPU时间。 该值以纳秒表示。 |
| `resultSize` | 此任务作为`TaskResult`传回给驱动程序的字节数。 |
| `jvmGCTime` | 执行此任务时`JVM`在垃圾收集中花费的时间。 该值以毫秒表示。 |
| `resultSerializationTime` | 序列化任务结果所花费的时间。 该值以毫秒表示 | 
| `memoryBytesSpilled` | 此任务溢出的内存中字节数。 |
| `diskBytesSpilled` | 此任务溢出的硬盘中字节数。 |
| `peakExecutionMemory` | 在随机，聚合和连接期间创建的内部数据结构使用的峰值内存。 此累加器的值应大约是此任务中创建的所有此类数据结构的峰值大小的总和。 对于`SQL`作业，这仅跟踪所有不安全的运算符和`ExternalSort`。 |
| `inputMetrics.*` | 与从`[[org.apache.spark.rdd.HadoopRDD]]`或持久数据中读取数据相关的度量标准。 |
| `.bytesRead` | 读取的总字节数。 |
| `.recordsRead` | 读取的记录总数。 |
| `outputMetrics.*` | 与外部写入数据相关的度量（例如，分布式文件系统），仅在具有输出的任务中定义。  | 
| `.bytesWritten` | 写入的总字节数。|
| `.recordsWritte` | 写入的记录总数。 | 
| `shuffleReadMetrics.*` |  与`shuffle`读取操作相关的度量标准。 |
| `.recordsRead` |  在`shuffele`操作中读取的记录数 |
| `.remoteBlocksFetched` | 在`shuffele`操作中获取的远程块数 |
| `.localBlocksFetched` | 在`shuffele`操作中获取的本地块数（与远程执行器读取相反）的数量 |
| `.totalBlocksFetched` | 在`shuffele`操作中获取的块数（本地和远程） | 
| `.remoteBytesRead` | 在`shuffele`操作中读取的字节数 |
| `.localBytesRead` | 在`shuffele`操作中获取的本地磁盘字节数（与远程执行器读取相反） | 
| `.totalBytesRead` | 在`shuffele`操作中获取的字节数（本地和远程） | 
| `.remoteBytesReadToDisk`| 在`shuffele`操作中读取到磁盘的远程字节数。 在随机读取操作中将大块提取到磁盘，而不是读入内存，这是默认行为。 |
| `.fetchWaitTime` | 等待远程`shuffle`块的任务所花费的时间。 这仅包括随机输入数据的时间阻塞。 例如，如果在任务尚未完成处理块A的情况下获取块B，则不认为块B在块B上被阻塞。该值以毫秒表示。 | 
| `shuffleWriteMetrics.*` | 与编写随机数据的操作相关的度量标准。 |
| `.bytesWritten` | 在`shuffle`操作中写入的字节数 |
| `.recordsWritten` | 在`shuffle`操作中写入的块数 | 
| `.writeTime` | 写入磁盘或缓冲区缓存时阻塞的时间。 该值以纳秒表示。 | 

### API 版本控制策略

这些`endpoint`已被强力版本化，以便更容易开发应用程序。特别是`Spark`保证：

*   `endpoint` 永远不会从一个版本中删除
*   任何给定 `endpoint` 都不会删除个别字段
*   可以添加新的 `endpoint`
*   可以将新字段添加到现有 `endpoint`
*   将来可能会在单独的 `endpoint` 添加新版本的 `api`（例如：`api/v2`）。新版本 _不_ 需要向后兼容。
*   `Api` 版本可能会被删除，但只有在至少一个与新的 `api` 版本共存的次要版本之后才可以删除。

请注意，即使在检查正在运行的应用程序的 `UI` 时，仍然需要 `applications/[app-id]`部分，尽管只有一个应用程序可用。例如：要查看正在运行的应用程序的作业列表，您可以访问 `http://localhost:4040/api/v1/applications/[app-id]/jobs`。这是为了在两种模式下保持路径一致。

# 指标

`Spark` 具有基于[Dropwizard Metrics Library](http://metrics.dropwizard.io/)的可配置 `metrics` 系统。这允许用户将 `Spark metrics` 报告给各种接收器，包括 `HTTP`，`JMX` 和 `CSV` 文件。`metrics` 系统是通过配置文件进行配置的，`Spark` 配置文件是 `Spark` 预计出现在 `$SPARK_HOME/conf/metrics.properties`上。可以通过`spark.metrics.conf` [配置属性](https://spark.apache.org/docs/latest/configuration.html#spark-properties)指定自定义文件位置。默认情况下，用于 `driver` 或 `executor metrics` 标准的根命名空间是 `spark.app.id` 的值。然而，通常用户希望能够跟踪 `driver` 和 `executors` 的应用程序的 `metrics`，这与应用程序 `ID`（即：`spark.app.id`）很难相关，因为每次调用应用程序都会发生变化。对于这种用例，可以为使用 `spark.metrics.namespace`配置属性的 metrics 报告指定自定义命名空间。例如，如果用户希望将度量命名空间设置为应用程序的名称，则可以将`spark.metrics.namespace`属性设置为像 `${spark.app.name}`这样的值。然后，该值会被 `Spark` 适当扩展，并用作度量系统的根命名空间。非 `driver`和 `executor` 的 `metrics` 标准永远不会以 `spark.app.id`为前缀，`spark.metrics.namespace`属性也不会对这些 `metrics` 有任何这样的影响。

`Spark` 的 `metrics` 被分解为与 `Spark` 组件相对应的不同 `_instances_`。在每个实例中，您可以配置一组报告汇总指标。目前支持以下实例：

*   `master`：`Spark standalone` 的 `master` 进程。
*   `applications`：主机内的一个组件，报告各种应用程序。
*   `worker`：`Spark standalone` 的 `worker` 进程。
*   `executor`：一个 `Spark executor`.
*   `driver`：`Spark driver` 进程（创建 `SparkContext` 的过程）。
*   `shuffleService`：`Spark shuffle` 服务.
*   `applicationMaster`:在`YARN`上运行时`Spark ApplicationMaster`。

每个实例可以报告为 0 或更多 _sinks_。`Sinks` 包含在 `org.apache.spark.metrics.sink`包中：

*   `ConsoleSink`：将 `metrics `信息记录到控制台。
*   `CSVSink`：定期将 `metrics` 数据导出到 `CSV` 文件。
*   `JmxSink`：注册在 `JMX` 控制台中查看的 `metrics`。
*   `MetricsServlet`：在现有的 `Spark UI` 中添加一个 `servlet`，以将数据作为 `JSON` 数据提供。
*   `GraphiteSink`：将 `metrics` 发送到 `Graphite` 节点。
*   `Slf4jSink`：将 `metrics` 标准作为日志条目发送到 `slf4j`。
*   `StatsdSink`:将指标发送到`StatsD`节点。

`Spark` 还支持由于许可限制而不包含在默认构建中的 `Ganglia` 接收器：

*   `GangliaSink`：向 `Ganglia` 节点或 `multicast` 组发送`metrics`。

要安装 `GangliaSink`，您需要执行 `Spark` 的自定义构建。_**请注意，通过嵌入此库，您将包括 [LGPL](http://www.gnu.org/copyleft/lesser.html)-licensed Spark包中的代码**_。对于 `sbt` 用户，在构建之前设置 `SPARK_GANGLIA_LGPL`环境变量。对于 `Maven` 用户，启用 `-Pspark-ganglia-lgpl` 配置文件。除了修改集群的 `Spark` 构建用户，应用程序还需要链接到 `spark-ganglia-lgpl` 工件。

`metrics` 配置文件的语法在示例配置文件 `$SPARK_HOME/conf/metrics.properties.template` 中定义。

# 高级工具

可以使用几种外部工具来帮助描述 `Spark job` 的性能：

*   集群范围的监控工具，例如 [Ganglia](http://ganglia.sourceforge.net/)可以提供对整体集群利用率和资源瓶颈的洞察。例如，`Ganglia `仪表板可以快速显示特定工作负载是否为磁盘绑定，网络绑定或 `CPU` 绑定。
*   操作系统分析工具，如 [dstat](http://dag.wieers.com/home-made/dstat/)，[iostat](http://linux.die.net/man/1/iostat) 和 [iotop](http://linux.die.net/man/1/iotop) 可以在单个节点上提供细粒度的分析。
*   `JVM` 实用程序，如 `jstack` 提供堆栈跟踪，`jmap` 用于创建堆转储，`jstat` 用于报告时间序列统计数据和 `jconsole` 用于可视化地浏览各种 JVM 属性对于那些合适的 `JVM` 内部使用是有用的。