
Debezium-PostgreSQL Connect  
=
----------------------------------------------------------------------
1.工作模式
-
  **PostgreSQL Connect**第一次连接数据库/集群时，读取所有模式的快照。完成后不断地将提交的更改操作，流式传输到Pg（9.6及更高版本），同时生成相应的操作事件。事物将记录在一个单独的**Kafka** 的**Topic**中。  

-------------------------------------------------
-

2.连接器及插件概述
-
  提取事务日志需要PostgreSQL的 [logical decoding](https://www.postgresql.org/docs/9.6/logicaldecoding-explanation.html)（逻辑解码）功能。安装logical decoding功能，才可以通过 [output plugin](https://www.postgresql.org/docs/9.6/logicaldecoding-output-plugin.html)（输出插件）提取 提交到事物日志中的操作以及处理这些操作。  

###2.1 Debezium的**PostgreSQL Connect**（后称：pg连接器）包含两部分：  

1.“逻辑解码输出插件” 二选一：  
-[Protobuf](https://github.com/debezium/postgres-decoderbufs) based  

-[wal2json](https://github.com/eulerto/wal2json)（
2019年一月起**AWS RDS**上的 **Pg9.6.10；Pg10.5；Pg11** 版本附带 **wal2json** 插件。  ）  

2.使用 [PostgreSQL’s streaming replication protocol](https://www.postgresql.org/docs/9.6/logicaldecoding-walsender.html)(Pg的流复制协议)，通过 [JDBC驱动](https://github.com/pgjdbc/pgjdbc) 读取Protobuf 插件生产的更改操作。  

###2.2 PG逻辑解码功能，有部分功能限制：  
1.不支持DDL更改，DDL操作事件不能报告给用户

2.[逻辑解码复制槽](https://blog.csdn.net/pg_hgdb/article/details/84105514) 只在 **master** 服务器上受支持，其他副本均无法运行。**master** 宕机后需手动重新调整连接器配置。   

###2.3 [PostgreSQL的逻辑解码输出插件安装文档](https://debezium.io/docs/install/postgres-plugins/) 

-----------------------------------------------------------
-
3.如何使用/配置 Amazon RDS 上的Pg
-
* rds.logical_replication 设置为 1。
* 以 master身份运行wal_level 验证是否设置成功；多区域复制可能存在差异。可参考[AWS用户指南](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithParamGroups.html)
* plugin.name Debezium参数设置为 wal2json。
* 需用主账户进行复制，因RDS不支持 replication 为其它账户设置权限。  

-------------------------------------------------------
-
4.插件配置细节、差异及关注点
-
Pg连接器与 逻辑解码插件 配合使用，可对Protobuf 或 JSON的变化进行编写。  

Debezium提供了基于Vanilla PostgreSQL服务器镜像的Docker镜像。官方文档建议使用[Docker-images](https://github.com/debezium/docker-images/tree/master/postgres/9.6)作为安装文档实示例。  

**插件之间的差异**（以下是现已发现的，仍存在未发现的）
  
* wal2json插件无法处理带引号的标识符  
* wal2json插件不会为没有主键的表发出事件
* wal2json插件不支持浮点类型的特殊值（NaN / infinity）
* wal2json应用于设置schema.refresh.mode连接器选项conlumns_diff_exclude_unchanged_toast;当接收包含未更改TOAST列的行 的更改操作时，该列的字段都不会包含在发出的更改操作的 after 结构中，因为**wal2json**的消息不包含此列的字段。在 **wal2json**的[issue 98#](https://github.com/eulerto/wal2json/issues/98)中有跟踪此情况。  

在此 [Java Class](https://github.com/debezium/debezium/blob/master/debezium-connector-postgres/src/test/java/io/debezium/connector/postgresql/DecoderDifferences.java) 中有记录所有最新差异。  

-----------
-

5.配置PostgreSQL服务器
-
插件安装后，服务器启动时就会加载插件，并定义复制插槽：  

MODULES：  

* shared_preload_libraries = 'decoderbufs,wal2json' **(1)**

REPLICATION：  

* wal_level = logical             **(2)**
* max_wal_senders = 1             **(3)**
* max_replication_slots = 1       **(4)**  
————————————————————————————————————————————————————  
**(1):** 告诉服务器它应该在启动时加载decoderbufs和wal2json逻辑解码插件（插件的名称在Protobuf和wal2json Makefiles 中设置  

**(2):**告诉服务器它应该使用带有预写日志的逻辑解码  

**(3):**告诉服务器它应该使用最多的1单独进程来处理WAL更改  

**(4):**告诉服务器它应该允许1为流式WAL更改创建最多的复制槽
Debezium需要在Debezium停机期间保留PostgreSQL的WAL。如果您的WAL保留太小而且中断时间太长，那么Debezium将无法在重启后恢复，因为它会遗漏部分数据更改。通常的指标是一个类似于启动期间抛出的错误：ERROR: requested WAL segment 000000010000000000000001 has already been removed。

发生这种情况时，有必要重新执行数据库的快照。我们还建议设置参数wal_keep_segments = 0。请按照PostgreSQL官方文档进行WAL保留的微调。  

###[PostgreSQL预写日志及配置官方文档](https://www.postgresql.org/docs/9.6/wal-configuration.html)  

------
-
6.PostgreSQL连接器的工作原理
-
##6.1快照  

###6.1.1快照模式一
多数PostgreSQL服务器被配置为不在WAL段中保留数据库的完整历史记录，因此Pg连接器无法通过读取WAL来查看数据库完整你是记录。Pg连接器默认下首次启动会执行数据库初始一致快照，步骤如下：  

* **1.**使用SERIALIZABLE，READ ONLY，DEFERRABLE隔离级别启动事务，以确保此事务中的所有后续读取都是针对单个一致版本的数据完成的。由于之后的任何更改数据INSERT，UPDATE以及DELETE其他客户端的操作不会是这个交易可见。  

* **2.**获取SHARE UPDATE EXCLUSIVE MODE每个受监控表的锁定，确保在快照发生时任何表都不会发生结构更改。注意，这些锁不会阻止表INSERTS，UPDATES和DELETES在操作过程中发生。  

* **3.**读取服务器事务日志中的当前位置。  

* **4.**扫描所有数据库表和模式，READ为每一行生成操作，并将该事件写入特定表的Kafka主题。  
* **5.**提交  

* **6.**完成  

如果连接器发生重新平衡，或在快照过程中 发生停止，则重启后开始新的快照。若完成初始化快照，即由第三步中读取的位置进行流式传输，确保监控完整。再次停止后，重启时由之前停止位置继续进行任务。  

**注意：**若发生长时间宕机PostgreSQL可能会清除时间较长的WAL段，并且会丢失最后的已知位置。此时重启后，Pg服务器不再具有起始点，且Pg连接器不能继续之前的工作。  

###6.1.2快照模式二  

此模式允许连接器始终执行快照。此行为告诉连接器在启动时始终执行快照，并在快照完成后继续执行上述序列中步骤3的流更改。此模式可用于已知某些WAL段已被删除且不再可用的情况，或者在新主节点升级后出现集群故障的情况下，以便连接器不会错过任何可能的更改这可能发生在新主机升级后但在新主机上重新启动连接器之前。  


###6.1.3快照模式三  

连接器从不执行快照。当以这种方式配置新连接器时，如果将继续从先前存储的偏移量进行流式更改，或者它将从首次在服务器上创建PostgreSQL逻辑复制插槽的时间点开始。请注意，只有当您知道所有感兴趣的数据仍然反映在WAL中时，此模式才有用。  

###6.1.4最终快照模式 （仅限初始）  

执行数据库快照，然后在流式传输任何其他更改之前停止。如果连接器已启动但在停止之前未完成快照，则连接器将重新启动快照进程并在快照完成后停止。


##6.2流式变化  

PostgreSQL连接器通常会从它连接的PostgreSQL服务器进行流更改。此机制依赖于PostgreSQL的复制协议，客户端在服务器的事务日志中，在某些位置（也称为Log Sequence Numbers简称LSN）中提交更改操作，从而接收服务器的更改操作。

服务器提交事务时，单独的服务器进程就会从逻辑解码插件调用回调函数。此函数会处理来自事务的更改操作，将它们转换为特定格式（在Debezium插件的情况下为Protobuf或JSON），并将它们写入输出流，然后客户端将使用它。

PostgreSQL连接器充当PostgreSQL客户端，当它接收到这些更改时，它会将事件转换为Debezium 创建，更新或删除包含事件的LSN位置的事件。PostgreSQL连接器将这些更改事件转发到Kafka Connect框架（在同一进程中运行），然后以相同的顺序将它们异步写入相应的Kafka Topic。Kafka Connect使用术语偏移量来表示Debezium为每个事件包含的源特定位置信息，Kafka Connect定期记录另一个Kafka Topic中的最新偏移量。

当Kafka Connect正常关闭时，它会停止连接器，将所有事件刷新到Kafka，并记录从每个连接器接收的最后一个偏移量。重启后，Kafka Connect会读取每个连接器的最后记录偏移量，并从该点开始连接器。PostgreSQL连接器使用每个更改事件中记录的LSN作为偏移量，以便在重新启动时连接器请求PostgreSQL服务器向其发送刚刚在该位置之后开始的事件。  

##6.3 Kafka Topic Names  

PostgreSQL连接器将单个表上的所有插入，更新和删除操作的事件写入单个Kafka主题。默认情况下，Kafka主题的名称采用serverName形式。schemaName。tableName，其中serverName是使用database.server.name配置属性指定的连接器的逻辑名称，schemaName是发生操作的数据库模式的名称，tableName是发生操作的数据库表的名称。

例，一个PostgreSQL安装有postgres数据库和一个inventory包含四个表的模式：products，products_on_hand，customers，和orders。如果监视此数据库的连接器的逻辑服务器名称为fulfillment，则连接器将生成有关这四个Kafka主题的事件：

* fulfillment.inventory.products

* fulfillment.inventory.products_on_hand

* fulfillment.inventory.customers

* fulfillment.inventory.orders

另，如果表不是特定模式的一部分，而是在默认的publicPostgreSQL模式中创建，那么Kafka主题的名称将是：

* fulfillment.public.products

* fulfillment.public.products_on_hand

* fulfillment.public.customers

* fulfillment.public.orders  

##6.4元信息  

recordPostgreSQL连接器生成的每个连接器除了数据库事件外，还有一些关于事件发生在服务器上的位置的元信息，源分区的名称以及需放置事件的Kafka主题和分区的名称：  

	"sourcePartition": {
         "server": "fulfillment"
	 },
	 "sourceOffset": {
         "lsn": "24023128",
         "txId": "555",
         "ts_usec": "1482918357011699"
	 },
	 "kafkaPartition": null  

PostgreSQL连接器仅使用1个Kafka Connect 分区，它将生成的事件放入1个Kafka分区。因此，sourcePartition将始终默认为database.server.name配置属性的名称，而kafkaPartition具有值null，这意味着连接器不使用特定的Kafka分区。

sourceOffset消息的一部分包含有关事件发生的服务器位置的信息：

* lsn表示PostgreSQL 日志序列号或offset在事务日志中

* txId 表示导致该事件的服务器事务的标识符

* ts_usec 表示自Unix Epoch以来的微秒数，作为提交事务的服务器时间  

##6.5 活动  

Debezium PostgreSQL连接器确保所有Kafka Connect 架构名称都是[有效的Avro架构名称](http://avro.apache.org/docs/current/spec.html#names)。这意味着逻辑服务器名称必须以拉丁字母或下划线（例如，[az，AZ，_]）开头，逻辑服务器名称中的其余字符以及架构和表名称中的所有字符必须是拉丁字母，数字或下划线（例如，[az，AZ，0-9，\ _]）。如果没有，则所有无效字符将自动替换为下划线字符。

当逻辑服务器名称，模式名称和表名称包含其他字符时，这可能会导致意外冲突，并且表全名之间唯一的区别字符无效，因此将替换为下划线。  

Debezium和Kafka Connect围绕连续的事件消息流进行设计，这些事件的结构可能会随着时间的推移而发生变化。对于消费者而言，这可能很难处理，因此为了使其变得简单，Kafka Connect使每个事件都自成一体。每个消息键和值都有两部分：模式和有效负载。模式描述了有效负载的结构，而有效负载包含实际数据。  

------------
-

7.错误解答及描述
-
Debezium是一个分布式系统，可捕获多个上游数据库中的所有更改，并且永远不会错过或丢失事件。当然，当系统名义上运作或小心管理时，Debezium只提供每次变更事件的一次交付。但是，如果确实发生了故障，那么系统仍然不会丢失任何事件，尽管在从故障中恢复时它可能会重复某些更改事件。因此，在这些异常情况下，Debezium（如Kafka）至少提供一次变更事件。  

##7.1  配置和启动错误  

启动时连接器将失败，在日志中报告错误/异常，并在连接器配置无效时，连接器无法使用指定的连接参数成功连接到PostgreSQL，或者连接器从以前重新启动时停止运行在PostgreSQL WAL中记录的位置（通过LSN值）和PostgreSQL不再具有该历史记录。

在这些情况下，错误将包含有关问题的更多详细信息以及可能的建议解决方法。在更正配置或解决PostgreSQL问题后，可以重新启动连接器。  

##7.2 PostgreSQL发生错误  

连接器运行后，如果连接的PostgreSQL服务器因任何原因变得不可用，连接器将失败并显示错误，连接器将停止。只需在服务器可用时重新启动连接器即可。

PostgreSQL连接器在外部存储最后处理的偏移量（以PostgreSQL log sequence number值的形式）。重新启动连接器并连接到服务器实例后，如果它具有先前存储的偏移量，它将要求服务器继续从该特定偏移量进行流式传输。但是，根据服务器配置，此特定偏移量可能在服务器的预写日志段中可用，也可能不可用。如果可用，则连接器将简单地恢复流更改而不会丢失任何内容。但是，如果此信息不可用，则连接器无法中继在其未联机时发生的更改  

##7.3 群集故障  

截至目前9.6，PostgreSQL 仅在主服务器上允许逻辑复制插槽，这意味着PostgreSQL连接器只能指向master数据库集群的活动状态。如果此计算机出现故障，只有在master提升新计时（安装了逻辑解码插件）后才能重新启动连接器并指向新服务器。

一个潜在的问题是，如果新服务器的升级和插件安装之间有足够大的延迟以及重新启动连接器，PostgreSQL服务器可能已经删除了一些WAL信息。如果发生这种情况，连接器将错过选择新主服务器之后和重新启动连接器之前发生的所有更改。  
[PostgreSQL的故障转移插槽](https://www.2ndquadrant.com/en/blog/failover-slots-postgresql/)  

##7.4 Kafka Connect流程停止  

若Kafka Connect以分布式模式运行，并且Kafka Connect进程正常停止，则在关闭该进程之前，Kafka Connect会将所有进程的连接器任务迁移到该组中的另一个Kafka Connect进程，以及新连接器任务将完全取消先前任务中断的位置。在连接器任务正常停止并在新进程上重新启动时，处理会有短暂的延迟。  

##7.5 Kafka Connect进程崩溃  

若Kafka Connector进程意外停止，那么它运行的任何连接器任务显然都会终止而不记录最近处理的偏移量。当Kafka Connect以分布式模式运行时，它将在其他进程上重新启动这些连接器任务。但是，PostgreSQL连接器将从先前进程记录的最后一个偏移量恢复，这意味着新的替换任务可能会生成一些在崩溃之前处理的相同更改事件。重复事件的数量取决于偏移刷新周期和崩溃之前的数据量变化。  

##7.6 Kafka不可用  

当连接器生成更改事件时，Kafka Connect框架使用Kafka生成器API在Kafka中记录这些事件。Kafka Connect还将以您在Kafka Connect工作人员配置中指定的频率定期记录这些更改事件中出现的最新偏移量。如果Kafka代理变得不可用，运行连接器的Kafka Connect工作进程将反复尝试重新连接到Kafka代理。换句话说，连接器任务将暂停，直到可以重新建立连接，此时连接器将从它们停止的位置恢复。  

##7.7 Pg Connect  

如果连接器正常停止，则可以继续使用数据库，并且将在PostgreSQL WAL中记录任何新的更改。重新启动连接器后，它将恢复最后一次停止的流更改，记录连接器停止时所做的所有更改的更改事件。

正确配置的Kafka群集能够实现大规模吞吐量。Kafka Connect是用Kafka编写的最佳实践，如果有足够的资源，也可以处理大量的数据库更改事件。因此，当连接器在一段时间后重新启动时，它很可能赶上数据库，但速度将取决于Kafka的功能和性能以及PostgreSQL中对数据所做的更改量。  

----------
-

#8 部署连接器  

如果你已经安装了Zookeeper，Kafka和[Kafka Connect](http://kafka.apache.org/documentation.html#connect)，那么使用Debezium的PostgreSQL连接器很容易。只需下载[连接器的插件存档](https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/0.9.5.Final/debezium-connector-postgres-0.9.5.Final-plugin.tar.gz)，将JAR解压缩到Kafka Connect环境中，然后将带有JAR的目录添加到Kafka Connect的类路径中。重新启动Kafka Connect进程以获取新的JAR。

如果你的东西是不可变的容器，那么请查看[Debezium的Docker镜像](https://hub.docker.com/u/debezium)，以获取Zookeeper，Kafka，PostgreSQL和Kafka Connect的PostgreSQL连接器，这些连接器已预先安装好并准备就绪。您甚至可以在Kubernetes和OpenShift上运行Debezium。

要使用连接器为特定的PostgreSQL服务器或集群生成更改事件，请执行以下操作：

安装逻辑解码插件

配置PostgreSQL服务器以支持逻辑复制

为PostgreSQL Connector创建配置文件，并使用Kafka Connect REST API将该连接器添加到Kafka Connect集群。

当连接器启动时，它将获取PostgreSQL服务器中数据库的一致快照，并开始流式更改，为每个插入，更新和删除的行生成事件。您还可以选择为模式和表的子集生成事件。（可选）忽略，屏蔽或截断敏感，过大或不需要的列。  


#[Debezium-PostgreSQL Connect官方文档链接](https://debezium.io/docs/connectors/postgresql/#events)  

更多示例代码以及关于数据类型的解释详见官方文档。