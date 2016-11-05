### [1.5 从早期版本升级](#upgrade)<a id="upgrade"></a>

#### [从0.8.x或0.9.x升级到0.10.0.0](#upgrade_10)<a id="upgrade_10"></a>

0.10.0.0 有一些潜在的[不兼容变更]((#upgrade_10_breaking))（在升级前请一定对其进行检查）和升级过程中性能下降的风险。遵照以下推荐的滚动升级方案，可以保证你在升级过程及之后都不需要停机并且没有性能下降。

注意：因为新的协议的引入，一定要先升级你的Kafka集群服务端然后再升级客户端。

**注意0.9.0.0版本客户端：** 因为在0.9.0.0版本客户端引入的一个bug，使得它不能与0.10.0.x版本中间件协作，这包括依赖ZooKeeper的客户端（原Scala上层（high-level）消费者和使用原消费者的MirrorMaker）。因此，0.9.0.0的客户端应该在服务端升级到0.10.0.0**之前**被升级到0.9.0.1上。这个步骤对于0.8.X和0.9.0.1的客户端不是不需要。

**采用滚动升级:**

1. 更新所有中间件的server.properties文件，添加如下配置：
  * inter.broker.protocol.version=CURRENT_KAFKA_VERSION (e.g. 0.8.2 or 0.9.0.0).
  * log.message.format.version=CURRENT_KAFKA_VERSION (参考 [**升级过程可能的性能影响**](#upgrade_10_performance_impact)了解这个配置项的作用)
2. 升级中间件。这个过程可以逐个中间件的去完成，只要将它下线然后升级代码然后重启即可。
3. 当整个集群都升级完成以后，再去处理协议版本问题，通过编辑inter.broker.protocol.version并设置为0.10.0.0即可。注意：此时还不应该去修改日志格式版本参数log.message.format.version-这参数只能在所有的客户端都升级到0.10.0.0之后去修改。
4. 逐个重启中间件让新的协议版本生效。
5. 当所有的消费者都升级到0.10.0以后，逐个中间件去修改log.message.format.version为0.10.0并重启。

**注意：** 如果你接受停机，你可简单将所有的中间件下线，升级代码然后再重启。这样它们应该都会默认使用新的协议。

**注意：** 修改协议版本并重启的工作你可以在升级中间件之后的任何时间进行，这个过程没必要升级代码后立即进行。

##### [升级0.10.0.0过程潜在的性能影响](#upgrade_10_performance_impact)<a id="upgrade_10_performance_impact"></a>

0.10.0版本的消息格式引入了一个新的timestamp字段并对压缩的消息使用了相对偏移量。磁盘的消息格式可以通过server.properties文件的log.message.format.version进行配置。默认的消息格式是0.10.0。对一个0.10.0之前版本的客户端，它只能识别0.10.0之前的消息格式。在这种情况下消息中间件可以将消息在响应给客户端之前转换成老的消息格式。但如此一来中间件就不能使用零拷贝传输了（zero-copy transfer）。根据Kafka社区的反馈包括，升级后这对性能的影响将会是CPU的使用率从20%提升到100%，这将迫使你必须立即升级所有的客户端到0.10.0.0版本来恢复性能表现。为了避免客户端升级到0.10.0.0之前的消息转换。你可以在升级中间件版本到0.10.0.0的过程中，将消息的格式参数og.message.format.version设置成0.8.2 或者 0.9.0版本。这样一来中间件依旧可以使用零拷贝传输来将消息发送到客户端。在所有的消费者升级以后，就可以修改中间件上消息格式版本到0.10.0，享受新消息格式带来的益处包括新引入的时间戳字段和更好的消息压缩。这个转换过程的支持是为了保证兼容性和支持少量未能及时升级到新版本客户端应用而存在的。如果想在一个即将超载的集群上来支持所有客户端的流量是不现实的。因此在消息中间升级以后但是主要的客户端还没有升级的时候应该尽可能的去避免消息转换。

对于已升级到0.10.0.0的客户端不存在这种性能上的负面影响。

**注意：**设置消息格式的版本，应该保证所有的已有的消息都是这个消息格式版本之下的版本。否则0.10.0.0之前的客户端可能出现故障。在实践中，一旦消息的格式被设置成了0.10.0之后就不应该把它再修改到早期的格式上，因为这可能造成0.10.0.0之前版本的消费者的故障。

**注意：**因为每个消息新时间戳字段的引入，生产者在发送小包消息可能出现因为负载上升造成的吞吐量的下降。同理，现在复制过程每个消息也要多传输8个比特。如果你的集群即将达到网络容量的瓶颈，这可能造成网卡打爆并因为超载引起失败和性能问题。

**注意：**如果你在生产者启动了消息压缩机制，你可能发现生产者吞吐量下降和/或中间件消息压缩比例的下降。在接受压缩过的消息时，0.10.0的中间避免重新压缩信息，这样原意是为了降低延迟提供吞吐量。但是在某些场景下，这可能降低生产者批处理数量，并引起吞吐量上更差的表现。如果这种情况发生了，用户可以调节生产者的linger.ms和batch.size参数来获得更好的吞吐量。另外生产者在使用snappy进行消息压缩时它用来消息压缩的buffer相比中间件的要小，这可能给磁盘上的消息的压缩率带来一个负面影响，我们计划将在后续的版本中将这个参数修改为可配置的。

##### [0.10.0.0潜在的不兼容修改](#upgrade_10_breaking)<a id="upgrade_10_breaking"></a>

* 从0.10.0.0开始，Kafka消息格式的版本号将用Kafka版本号表示。例如，消息格式版本号0.9.0表示最高Kafka 0.9.0支持的消息格式。
* 引入了0.10.0版本消息格式并作为默认配置。它引入了一个时间戳字段并在压缩消息中使用相对偏移量。
* 引入了ProduceRequest/Response v2并用作0.10.0消息格式的默认支持。
* 引入了FetchRequest\/Response v2并用作0.10.0消息格式的默认支持。
* 接口MessageFormatter从`def writeTo(key: Array[Byte], value: Array[Byte], output: PrintStream)`变更为`def writeTo(consumerRecord: ConsumerRecord[Array[Byte], Array[Byte]], output: PrintStream)`
* 接口MessageReader从`def readMessage(): KeyedMessage[Array[Byte], Array[Byte]]`变更为`def readMessage(): ProducerRecord[Array[Byte], Array[Byte]]`
* MessageFormatter的包从`kafka.tools`变更为`kafka.common`
* MessageReader的包`kafka.tools`变更为`kafka.common`
* MirrorMakerMessageHandler不再暴露`handle(record: MessageAndMetadata[Array[Byte], Array[Byte]])`方法，因为它从没有被调用过.
* 0.7 KafkaMigrationTool不再和Kafka打包。如果你需要从0.8迁移到0.10.0，请先迁移到0.8然后在根据文档升级过程完成从0.8到0.10.0的升级。
* 新的消费者规范化了API来使用`java.util.Collection`作为序列类型方法参数。现存的代码可能需要进行修改来实现0.10.0版本客户端库的协作。
* LZ4压缩消息的处理变更为使用互操作框架规范（LZ4f v1.5.1）（interoperable framing specification）。为了保证对老客户端的兼容，这个变更只应用于0.10.0或之后版本的消息格式上。v0/v1 (消息格式 0.9.0)生产或者拉取LZ4压缩消息的客户端将依旧使用0.9.0的框架实现。使用Produce/Fetch protocols v2协议的客户端及之后客户端应该使用互操作LZ4f框架。互操作（interoperable）LZ4类库列表可以在这里找到http://www.lz4.org/

##### [0.10.0.0值得关注的变更](#upgrade_10_notable)<a id="upgrade_10_notable"></a>

* 从Kafka 0.10.0.0开始，一个称为**Kafka Streams**的新客户端类库被引入，用于对存储于Kafka Topic的数据进行流处理。因为上文介绍的新消息格式变更，新的客户端类库只能与0.10.x或以上版本的中间件协作。详细信息请参[考此章节](http://kafka.apache.org/documentation.html#streams_overview)。
* 新消费者的默认配置参数`receive.buffer.bytes`现在变更为64K。

* 新的消费者暴露配置参数`exclude.internal.topics`限制内部话题topic(例如消费者偏移量topic)被意外的包括到正则表达式订阅之中。默认启动。

* 不推荐再使用原来的Scala生产者。用户应该尽快迁移他们的代码到kafka-clients Jar包中Java生产者上。
* 新的消费者API被认定为进入稳定版本（stable）

#### [从0.8.0, 0.8.1.X或0.8.2.X升级到0.9.0.0](#upgrade_9)<a id="upgrade_9"></a>

0.9.0.0 存在一些潜在的[不兼容变更](#upgrade_9_breaking) (在升级之前请一定查阅)和中间件间部协议与上一版本也发生了的变更。这意味升级中间件和客户端可能发生于老版本的不兼容情况。在升级客户端之前一定要先升级Kafka集群这一点很重要。如果你使用了MirrorMaker下游集群也应该对其先进行升级。

**滚动升级：**

1. 升级所有中间的server.properties文件，添加如下配置：inter.broker.protocol.version=0.8.2.X
2. 升级中间件。这个过程可以逐个中间件的去完成，只要将它下线然后升级代码然后重启即可。
3. 在整个集群升级完成以后，设置协议版本修改inter.broker.protocol.version设置成0.9.0.0
4. 逐个重启中间件让新的协议版本生效。

**注意：** 如果你接受停机，你可简单将所有的中间件下线，升级代码然后再重启。这样它们应该都会默认使用新的协议。

**注意：** 修改协议版本并重启的工作你可以在升级中间件之后的任何时间进行，这个过程没必要升级代码后立即进行。

##### [0.9.0.0潜在的不兼容变更](#upgrade_9_breaking)<a id="upgrade_9_breaking"></a>

* 不再支持Java 1.6。
* 不再支持Scala 2.9。
* 1000以上的Broker ID被默认保留用于自动分配broker id。如果你的集群现在有broker id大于此数值应该注意修改reserved.broker.max.id配置。
* 配置参数replica.lag.max.messages was removed。分区领导判定副本是否同步不再考虑延迟的消息。
* 配置参数replica.lag.time.max.ms现在不仅仅代表自副本最后拉取请求到现在的时间间隔，它也是副本最后完成同步的时间。副本正在从leader拉取消息，但是在replica.lag.time.max.ms时间内还没有完成最后一条信息的拉取，它也将被认为不再是同步状态。
* 压缩话题（Compacted topics）不再接受没有key的消息，如果消费者尝试将抛出一个异常。在0.8.x版本，一个不包含key的消息会引起日志压缩线程（log compaction thread）异常并退出（造成所有压缩话题的压缩工作中断）。
* MirrorMaker不再支持多个目标集群。这意味着它能只能接受一个--consumer.config参数配置。为了镜像多个源集群，你至少要为每个源集群配置一个MirrorMaker实例，每个实例配置他们的消费者信息。
* 原打包在_org.apache.kafka.clients.tools.*_的工具类移动到了_org.apache.kafka.tools.*_里面。所有包含的脚本能正常的使用，只有自定义代码直接import了那些类会受到影响。
* 默认的Kafka JVM 性能配置(KAFKA_JVM_PERFORMANCE_OPTS)在kafka-run-class.sh发生了变更。
* kafka-topics.sh脚本(kafka.admin.TopicCommand)在异常退出情况exit code变更为非零。
* 当topic names引起指标冲突时kafka-topics.sh脚本(kafka.admin.TopicCommand)将打印一个warn，这是由topic name使用了'.'或者'_'并且在用例中实际发生了冲突引起的。
* kafka-console-producer.sh脚本将默认使用新的生产者实例而不是老的，用户希望使用老的生产者必须指定'old-producer'。
* 默认所有的命令行工具将打印所有的日志信息到stderr而不是stdout。

##### [0.9.0.1值得关注的变更](#upgrade_901_notable)<a id="upgrade_901_notable"></a>

* 新的broker id生成功能可以通过broker.id.generation.enable设置为flase关闭。
* 配置参数log.cleaner.enable现在默认值为true。这意味着使用cleanup.policy=compact的话题将不再默认被压缩，并且一个128M的堆会被分配给清理进程（cleaner process），这个大小由log.cleaner.dedupe.buffer.size决定。在使用了压缩话题（compacted topics）时你可能需要评估你的log.cleaner.dedupe.buffer.size和其它log.cleaner配置。
* 新的消费者参数fetch.min.bytes默认配置为1

##### 0.9.0.0中弃用

* 通过kafka-topics.sh脚本修改topic信息已经弃用。今后请使用kafka-configs.sh完成此功能。
* kafka-consumer-offset-checker.sh（kafka.tools.ConsumerOffsetChecker）已经弃用。今后请用kafka-consumer-groups.sh完成此功能。
* The kafka.tools.ProducerPerformance class has been deprecated. Going forward, please use org.apache.kafka.tools.ProducerPerformance for this functionality \(kafka-producer-perf-test.sh will also be changed to use the new class\).
* kafka.tools.ProducerPerformance类已经弃用。今后使用org.apache.kafka.tools.ProducerPerformance完成此功能(kafka-producer-perf-test.sh也将变更为使用新的类)。
* 生产者配置block.on.buffer.full已经被弃用并在后续版本中移除。当前它的默认值已经被修改为false。Kafka生产者不再抛出BufferExhaustedException取而代之使用max.block.ms来阻塞，在阻塞超时以后将抛出TimeoutException。如果block.on.buffer.full属性被明确配置为true，它将设置max.block.ms为Long最大值（Long.MAX_VALUE）并且metadata.fetch.timeout.ms配置将不再被参考。
#### [从0.8.1升级到0.8.2](#upgrade_82)<a id="upgrade_82"></a>

0.8.2与0.8.1完全兼容。升级过程可以通过简单的逐个下线、升级代码、重启完成。

#### [从0.8.0升级到0.8.1](#upgrade_81)<a id="upgrade_81"></a>

0.8.1与0.8.0完全兼容。升级过程可以通过简单的逐个下线、升级代码、重启完成。

#### [从0.7升级](#upgrade_7)<a id="upgrade_7"></a>

0.7的发布版本与心得发布不兼容。核心的变更涉及到了API、ZooKeeper数据结构、协议以及实现复制的配置（之前0.7缺失的）。从0.7升级到后续的版本需要使用[特殊的工具](https://cwiki.apache.org/confluence/display/KAFKA/Migrating+from+0.7+to+0.8)来完成迁移。迁移过程可以实现不停机。
