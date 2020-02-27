# Spark 安全

## Spark安全：你要知道的事情

`Spark`中的安全性默认为`OFF`。 这可能意味着您很容易受到默认攻击。 `Spark`支持多种部署类型，每种类型都支持不同级别的安全性。 并非所有部署类型在所有环境中都是安全的，默认情况下都不是安全的。 请务必评估您的环境，`Spark`支持的内容，并采取适当措施来保护您的`Spark`部署。

存在许多不同类型的安全问题。 `Spark`并不一定能防范所有事情。 下面列出了`Spark`支持的一些内容。 另请查看部署文档，了解您用于部署特定设置的部署类型。 任何没有记录的东西，`Spark`都不支持。

## Spark RPC(Spark进程的通信协议)

### 认证

`Spark`目前支持使用共享密钥对`RPC`通道进行身份验证。可以通过设置`spark.authenticate`配置参数来打开身份验证。

用于生成和分发共享密钥的确切机制是特定于部署的。

对于`YARN`上的`Spark`和本地部署，`Spark`将自动处理生成和分发共享密钥。每个应用程序将使用唯一的共享密钥。在`YARN`的情况下，此功能依赖于启用`YARN RPC`加密来分发秘密是安全的。

对于其他资源管理器，必须在每个节点上配置`spark.authenticate.secret`。所有守护进程和应用程序将共享此秘密，因此此部署配置不如上述安全，特别是在考虑多租户群集时。在此配置中，具有秘密的用户可以有效地模仿任何其他用户。

`Rest`提交服务器和`MesosClusterDispatcher`不支持身份验证。您应确保对`REST API`和`MesosClusterDispatcher`的所有网络访问（默认情况下分别为端口6066和7077）仅限于受信任提交作业的主机。

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `spark.authenticate` | `false` | `Spark`是否验证其内部连接 |
| `spark.authenticate.secret` | `None` | 密钥使用身份验证。 请参阅上文，了解何时应设置此配置。 |

### 加密

`Spark`支持基于`AES`的`RPC`连接加密。 要启用加密，还必须启用并正确配置`RPC`身份验证。 `AES`加密使用[`Apache Commons Crypto`](https://commons.apache.org/proper/commons-crypto/)库，`Spark`的配置系统允许高级用户访问该库的配置。

还支持基于`SASL`的加密，但应将其视为已弃用。 与早于`2.2.0`的`Spark`版本的`shuffle`服务通信时仍然需要它。

下表介绍了可用于配置此功能的不同选项。 

Spark 当前支持使用 shared secret（共享密钥）来 authentication（认证）。可以通过配置 `spark.authenticate` 参数来开启认证。这个参数用来控制 Spark 的通讯协议是否使用共享密钥来执行认证。这个认证是一个基本的握手，用来保证两侧具有相同的共享密钥然后允许通讯。如果共享密钥无法识别，他们将不能通讯。创建共享密钥的方法如下:

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `spark.network.crypto.enabled` | `false` | 启用基于`AES`的`RPC`加密，包括`2.2.0`中添加的新身份验证协议。 |
| `spark.network.crypto.keyLength` | `128` | 要生成的加密密钥的位长度。 有效值为128,192和256。 |
| `spark.network.crypto.keyFactoryAlgorithm` | `PBKDF2WithHmacSHA1` | 生成加密密钥时使用的密钥工厂算法。 应该是正在使用的`JRE`中的`javax.crypto.SecretKeyFactory`类支持的算法之一。 |
| `spark.network.crypto.config.*` | `None` | `commons-crypto`库的配置值，例如要使用的密码实现。 配置名称应该是没有`commons.crypto`前缀的`commons-crypto`配置的名称。 | 
| `spark.network.crypto.saslFallback` | `true` | 如果使用`Spark`的内部机制验证失败，是否回退到`SASL`身份验证。 当应用程序连接到不支持内部`Spark`身份验证协议的旧`shuffle`服务时，这很有用。 在`shuffle`服务端，禁用此功能将阻止旧客户端进行身份验证。 | 
| `spark.authenticate.enableSaslEncryption` | `false` | 启用基于`SASL`的加密通信。 |
| `spark.authenticate.enableSaslEncryption` | `false` | 使用`SASL`身份验证禁用端口的未加密连接。 这将拒绝启用了身份验证的客户端的连接，但不请求基于`SASL`的加密。 | 

## 本地存储加密

`Spark`支持加密写入本地磁盘的临时数据。 这包括`shuffle`文件，`shuffle`溢出和存储在磁盘上的数据块（用于缓存和广播变量）。 它不包括使用诸如`saveAsHadoopFile`或`saveAsTable`等`API`的应用程序生成的加密输出数据。 它也可能不包括用户明确创建的临时文件。

以下设置包括对写入磁盘的数据启用加密：

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `spark.io.encryption.enabled` | `false` | 启用本地磁盘`I/O`加密。 目前除了`Mesos`之外的所有模式都支持。 强烈建议在使用此功能时启用`RPC`加密。 | 
| `spark.io.encryption.keySizeBits` | 128 | `IO`加密密钥大小，以位为单位。 支持的值为128,192和256。 |
| `spark.io.encryption.keygen.algorithm` | `HmacSHA1` | 生成`IO`加密密钥时使用的算法。 `Java Cryptography`体系结构标准算法名称文档的`KeyGenerator`部分中描述了支持的算法。 |
| `spark.io.encryption.commons.config.*` | `None` | `commons-crypto`库的配置值，例如要使用的密码实现。 配置名称应该是没有`commons.crypto`前缀的`commons-crypto`配置的名称。 |

## Web UI

### 认证和授权

使用`javax servlet`过滤器为`Web UI`启用身份验证。您将需要一个实现要部署的身份验证方法的筛选器。 `Spark`不提供任何内置的身份验证筛选器。

当存在身份验证筛选器时，`Spark`还支持对`UI`的访问控制。每个应用程序都可以配置自己独立的访问控制列表（`ACL`）。 `Spark`区分“视图”权限（允许谁查看应用程序的UI）和“修改”权限（谁可以执行诸如在正在运行的应用程序中终止作业）。

可以为用户或组配置`ACL`。配置条目接受以逗号分隔的列表作为输入，这意味着可以为多个用户或组授予所需的权限。如果您在共享群集上运行并且有一组管理员或开发人员需要监视他们可能尚未启动的应用程序，则可以使用此方法。添加到特定`ACL`的通配符
`(*)`表示所有用户都具有相应的权限。默认情况下，只有提交应用程序的用户才会添加到`ACL`中。

通过使用可配置的组映射提供程序来建立组成员身份。映射器使用`spark.user.groups.mapping`配置选项进行配置，如下表所述。

以下选项控制`Web UI`的身份验证：

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `spark.ui.filters` | `None` | 有关如何配置筛选器的信息，请参阅[Spark UI配置](https://spark.apache.org/docs/latest/configuration.html#spark-ui)。 |
| `spark.acls.enable` | `false` | 是否应启用`UI ACL`。 如果启用，则会检查用户是否具有查看或修改应用程序的访问权限。 请注意，这需要对用户进行身份验证，因此如果未安装身份验证筛选器，则此选项不会执行任何操作。 |
| `spark.admin.acls` | `None` | 以逗号分隔的用户列表，可以查看和修改对`Spark`应用程序的访问权限。 |
| `spark.admin.acls.groups` | `None` | 以逗号分隔的组列表，可以查看和修改对`Spark`应用程序的访问权限。 |
| `spark.modify.acls` |`None` | 以逗号分隔的用户列表，具有对`Spark`应用程序的修改权限。 |
| `spark.modify.acls.groups` | `None` | 以逗号分隔的组列表，具有对`Spark`应用程序的修改权限。 | 
| `spark.ui.view.acls` | `None` | 以逗号分隔的用户列表，具有对`Spark`应用程序的视图访问权限。 |
| `spark.ui.view.acls.groups` | `None` | 以逗号分隔的组列表，具有对`Spark`应用程序的视图访问权限。 |
| `spark.user.groups.mapping` | `org.apache.spark.security.ShellBasedGroupsMappingProvider` | 用户的组列表由特征`org.apache.spark.security.GroupMappingServiceProvider`定义的组映射服务确定，该服务可由此属性配置。默认情况下，使用基于`Unix shell`的实现，该实现从主机`OS`收集此信息。注意：此实现仅支持基于`Unix/Linux`的环境。 目前不支持`Windows`环境。 但是，通过实现上述特征可以支持新的平台/协议。 |

在`YARN`上，视图和修改`ACL`在提交应用程序时提供给`YARN`服务，并通过`YARN`接口控制具有相应权限的人员。

### Spark历史服务器ACLs

使用`servlet`过滤器，可以使用与常规应用程序相同的方式启用`SHS Web UI`的身份验证。

要在`SHS`中启用授权，可以使用一些额外的选项：

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `spark.history.ui.acls.enable` | `false` | 指定是否应检查`ACL`以授权用户查看历史记录服务器中的应用程序。 如果启用，则无论各个应用程序为`spark.ui.acls.enable`设置了什么，都会执行访问控制检查。 应用程序所有者将始终有权查看自己的应用程序，并且运行应用程序时通过`spark.ui.view.acls.groups`指定的`spark.ui.view.acls`和组指定的任何用户也将有权查看该应用程序。 如果禁用，则不会对通过历史记录服务器提供的任何应用程序UI进行访问控制检查。 |
| `spark.history.ui.admin.acls` | `None` | 逗号分隔的用户列表，具有对历史服务器中所有`Spark`应用程序的视图访问权限。 |
| `spark.history.ui.admin.acls.groups` | `None` | 逗号分隔的组列表，具有对历史服务器中所有`Spark`应用程序的视图访问权限。 |

`SHS`使用相同的选项将组映射提供程序配置为常规应用程序。 在这种情况下，组映射提供程序将由`SHS`应用于所有`UI`服务器，并且将忽略各个应用程序配置。

### SSL配置

`SSL`的配置按层次结构组织。 用户可以配置默认`SSL`设置，这些设置将用于所有支持的通信协议，除非它们被特定于协议的设置覆盖。 这样，用户可以轻松地为所有协议提供通用设置，而无需单独配置每个协议。 下表描述了`SSL`配置命名空间：

| 配置命名空间 | 组件 | 
| --- | --- |
| `spark.ssl` | 默认的`SSL`配置。 除非在命名空间级别显式覆盖，否则这些值将应用于下面的所有命名空间。 |
| `spark.ssl.ui`  | `Spark`应用`Web UI` | 
| `spark.ssl.standalone` | `Standalone Master/Slave Web UI`|
| `spark.ssl.historyServer` | 历史服务器`Web UI` |

可以在下面找到可用`SSL`选项的完整细分。 `${ns}`占位符应替换为上述命名空间之一。

| 属性名 | 默认值 | 含义 |
|---|---|---|
| `${ns}.enabled` | `false` | 启用`SSL`。 启用时，需要`${ns}.ssl.protocol`。 | 
| `${ns}.port` | `None` | `SSL`服务将侦听的端口。必须在特定命名空间配置中定义端口。 读取此配置时，将忽略默认命名空间。未设置时，`SSL`端口将从同一服务的非`SSL`端口派生。 值"0"将使服务绑定到临时端口。 |
| `${ns}.enabledAlgorithms` | `None` | 以逗号分隔的密码列表。 `JVM`必须支持指定的密码。参考协议列表可以在`Java`安全指南的“JSSE密码套件名称”部分找到。 可以在[此页面](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#ciphersuites)找到`Java 8`的列表。注意：如果未设置，将使用`JRE`的默认密码套件。 |
| `${ns}.keyPassword` | `None` | 密钥库中私钥的密码。 |
| `${ns}.keyStore` | `None` | 密钥库文件的路径。 该路径可以是绝对路径，也可以是相对于启动进程的目录。 | 
| `${ns}.keyStorePassword` | `None` | 密钥库存储的密码。 |
| `${ns}.keyStoreType` | `JKS` |  秘钥库存储的类型。 |
| `${ns}.protocol` | `None` | 要使用的`TLS`协议。 `JVM`必须支持该协议。协议的参考列表可以在`Java`安全指南的`Additional JSSE Standard Names`部分找到。 对于`Java 8`，可以在[此页](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#jssenames)面找到该列表。 |
| `${ns}.needClientAuth` | `false` | 是否要求客户端身份验证 |
| `${ns}.trustStore` | `None` | 信任库文件的路径。 该路径可以是绝对路径，也可以是相对于启动进程的目录。 | 
| `${ns}.trustStorePassword` | `None` | 信任存储的密码。 |
| `${ns}.trustStoreType` | `JKS` | 信任存储的类型。 |

`Spark`还支持从`Hadoop`凭据提供程序中检索`${ns}.keyPassword`,`${ns}.keyStorePassword`和`${ns}.trustStorePassword`。 用户可以将密码存储到凭证文件中，并使其可以被不同的组件访问，例如：
```
hadoop credential create spark.ssl.keyPassword -value password \
    -provider jceks://hdfs@nn1.example.com:9001/user/backup/ssl.jceks
```
要配置凭据提供程序的位置，请在`Spark`使用的`Hadoop`配置中设置`hadoop.security.credential.provider.path`配置选项，如：
```
  <property>
    <name>hadoop.security.credential.provider.path</name>
    <value>jceks://hdfs@nn1.example.com:9001/user/backup/ssl.jceks</value>
  </property>
```
或者通过`SparkConf``spark.hadoop.hadoop.security.credential.provider.path=jceks：//hdfs@nn1.example.com：9001/user/backup/ssl.jceks`。

### 准备秘钥存储

密钥库可以通过`keytool`程序生成。 这个`Java 8`工具的参考文档就在[这里](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html)。 为`Spark Standalone`部署模式配置密钥库和信任库的最基本步骤如下：

- 为每个节点生成密钥对
- 将密钥对的公钥导出到每个节点上的文件
- 将所有导出的公钥导入单个信任库
- 将信任库分发到群集节点

#### YARN模式

要向以群集模式运行的驱动程序提供本地信任库或密钥库文件，可以使用`--files`命令行参数（或等效的`spark.files`配置）将它们与应用程序一起分发。 这些文件将放在驱动程序的工作目录中，因此`TLS`配置应该只引用没有绝对路径的文件名。

以这种方式分发本地密钥存储可能需要在`HDFS`（或集群使用的其他类似分布式文件系统）中暂存文件，因此建议在配置安全性的基础文件系统时（例如，通过启用身份验证和线路加密））。

#### Standalone模式

用户需要为`master`和`worker`提供密钥库和配置选项。 必须通过在`SPARK_MASTER_OPTS和SPARK_WORKER_OPTS`环境变量中附加适当的`Java`系统属性来设置它们，或者仅在`SPARK_DAEMON_JAVA_OPTS`中附加它们。

用户可以允许执行者使用从工作进程继承的`SSL`设置。 这可以通过将`spark.ssl.useNodeLocalConf`设置为`true`来完成。 在这种情况下，不使用用户在客户端提供的设置。

#### Mesos模式

`Mesos 1.3.0`和更新版本支持`Secrets`原语作为基于文件和基于环境的秘密。 `Spark`允许分别使用`spark.mesos.driver.secret.filenames和spark.mesos.driver.secret.envkeys`来指定基于文件和基于环境变量的机密。

根据秘密存储后端，秘密可以通过引用或值分别通过`spark.mesos.driver.secret.names和spark.mesos.driver.secret.values`配置属性传递。

参考类型机密由秘密商店提供并通过名称引用，例如`/mysecret`。 值类型机密在命令行上传递并转换为适当的文件或环境变量。 


## HTTP安全头部

`Apache Spark`可以配置为包含`HTTP`标头，以帮助防止跨站点脚本（`XSS`），跨框架脚本（`XFS`），`MIME`嗅探，以及强制`HTTP`严格传输安全性。

| 属性 | 默认值 | 含义 |
| --- |---|---|
| `spark.ui.xXssProtection` | `1; mode=block` | `HTTP` `X-XSS-Protection`响应头的值。 您可以从下面选择适当的值：`0`（禁用`XSS`过滤）<br>`1`（启用`XSS`过滤。如果检测到跨站点脚本攻击，浏览器将清理该页面。）<br>`1; mode = block`（启用`XSS`过滤。如果检测到攻击，浏览器将阻止呈现页面。） |
| `spark.ui.xContentTypeOptions.enabled` | `true` |  启用后，`X-Content-Type-Options` `HTTP`响应标头将设置为`nosniff`。 |
| `spark.ui.strictTransportSecurity` | `None` | `HTTP`严格传输安全性（`HSTS`）响应标头的值。 您可以从下面选择适当的值并相应地设置`expire-time`。 此选项仅在启用`SSL/TLS`时使用`max-age=<expire-time>`<br>`max-age=<expire-time>;;includeSubDomains`<br>`max-age=<expire-time>;preload` |


## 为网络安全配置端口

一般而言，`Spark`群集及其服务未部署在公共`Internet`上。 它们通常是私有服务，只能在部署`Spark`的组织的网络中访问。 对`Spark`服务使用的主机和端口的访问应限于需要访问服务的源主机。

以下是`Spark`用于通信的主要端口以及如何配置这些端口。

### 仅Standalone模式适用

| From | To | 默认端口 | 目的| 配置设置 | 注意 |
| --- | --- | --- | --- | --- | --- |
| `Browser` | `Standalone Master` | `8080` | `Web UI` | `spark.master.ui.port /`<br>`SPARK_MASTER_WEBUI_PORT` | `Jetty-based`. 仅Standalone模式适用. |
| `Browser` | `Standalone Worker` | `8081` | `Web UI` | `spark.worker.ui.port /`<br>`SPARK_WORKER_WEBUI_PORT` | Jetty-based. 仅Standalone模式适用. |
| `Driver` /<br>`Standalone Worker` | `Standalone Master` | `7077` | 提交到`cluster`/<br>`Join cluster` | `SPARK_MASTER_PORT` | 设置为"0"随机选择端口. 仅Standalone模式适用. |
| `External Service` | `Standalone Master`	| `6066` | 	通过`REST API`提交到`cluster`  | `spark.master.rest.port` | 	使用`spark.master.rest.enabled`来启用/禁用这个服务. 仅Standalone模式适用. | 
| `Standalone Master` | `Standalone Worker` | (随机) | `Schedule executors` | `SPARK_WORKER_PORT` | 设置为"0"随机选择端口. 仅Standalone模式适用. |

### 所有cluster管理器

| From | To | 默认端口| 目的 | 配置设置 | 注意 |
| --- | --- | --- | --- | --- | --- |
| `Browser` | `Application` | `4040` | `Web UI` | `spark.ui.port` | `Jetty-based` |
| `Browser` | `History Server` | `18080` | `Web UI` | `spark.history.ui.port` | `Jetty-based` |
| `Executor`/<br>`Standalone Master` | `Driver` | `(random)` | 连接到应用/<br>通知执行者状态更改 | `spark.driver.port` | 设置为"0"随机选择端口. |
| `Executor` /<br>`Driver` | `Executor`/<br>`Driver` | `(random)` | `Block Manager port` | `spark.blockManager.port` | 通过`ServerSocketChannel`的原始套接字 |

## Kerberos

`Spark`支持在使用`Kerberos`进行身份验证的环境中提交应用程序。在大多数情况下，`Spark`在对`Kerberos`感知服务进行身份验证时依赖于当前登录用户的凭据。可以通过使用`kinit`等工具登录配置的`KDC`来获取此类凭据。

在与基于`Hadoop`的服务交谈时，`Spark`需要获取委托令牌，以便非本地进程可以进行身份​​验证。 `Spark`支持`HDFS`和其他`Hadoop`文件系统，`Hive`和`HBase`。

使用`Hadoop`文件系统（如`HDFS`或`WebHDFS`）时，`Spark`将获取托管用户主目录的服务的相关令牌。

如果`HBase`位于应用程序的类路径中，并且`HBase`配置已启用`Kerberos`身份验证（`hbase.security.authentication = kerberos`），则将获取`HBase`令牌。

类似地，如果`Hive`在类路径中，则将获得`Hive`令牌，并且配置包括远程`Metastore`服务的URI（`hive.metastore.uris`不为空​​）。

目前仅在`YARN`和`Mesos`模式下支持委托令牌支持。有关更多信息，请参阅特定于部署的页面。

以下选项为此功能提供了更细粒度的控制：

| 属性 | 默认值 | 含义 |
| --- | --- |--- |
| `spark.security.credentials.${service}.enabled` | `true` | 控制在启用安全性时是否获取服务凭据。 默认情况下，在配置这些服务时会检索所有受支持服务的凭据，但如果它与正在运行的应用程序发生某种冲突，则可以禁用该行为。 |

### 长期运行的程序

如果运行时间超过其需要访问的服务中配置的最大委托令牌生存期，则长时间运行的应用程序可能会遇到问题。

在`YARN`模式下运行时，`Spark`支持为这些应用程序自动创建新令牌。 需要使用`--principal`和`--keytab`参数通过`spark-submit`命令将`Kerberos`凭据提供给`Spark`应用程序。

提供的密钥表将通过`Hadoop`分布式缓存复制到运行`Application Master`的计算机。 出于这个原因，强烈建议至少使用加密来保护`YARN`和`
HDFS`。

## 事件日志

如果您的应用程序正在使用事件日志记录，则应使用适当的权限手动创建事件日志所在的目录（`spark.eventLog.dir`）。 要保护日志文件，应将目录权限设置为`drwxrwxrwxt`。 目录的所有者和组应对应于运行`Spark History Server`的超级用户。

这将允许所有用户写入目录，但会阻止非特权用户读取，删除或重命名文件，除非他们拥有该文件。 `Spark`将使用权限创建事件日志文件，以便只有用户和组具有读写访问权限。
将使用提供的凭据定期更新Kerberos登录，并将创建支持的新委派令牌。