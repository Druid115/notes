## Kafka 生产者

### 消息发送流程

生产者中有两个线程 — mian 线程和 sender 线程。main 线程中创建了一个双端队列 RecordAccumulator，main 线程将消息发送给 RecordAccumulator 队列，sender 线程不断从 RecordAccumulator 中拉取消息发送到 Kafka Broker。



### 分区策略
1. 指明 partition 的情况下，直接使用指定的值。
2. 没有指明 partition 但有 key 的情况下，将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值；
3. 既没有指明 partition 又没有 key 的情况下，采用 Sticky Partition（黏性分区器），会随机选择一个分区，并尽可能一直使用该分区，待该分区的 batch 已满，再随机选择一个另外的分区进行使用。



### 吞吐量

可以通过以下参数进行调整：

- batch.size：批次大小，默认16k。设置太小会导致频繁网络请求，吞吐量下降；设置太大会导致数据发送延迟并加大内存缓冲区的压力。
- linger.ms：等待时间，默认 0，消息会立即被发送。
- compression.type：压缩方式，有 none、gzip、snappy、lz4 和 zstd，默认 none。
- buffer.memory： RecordAccumulator 缓冲区大小，默认 32m。设置合理的值，保证缓冲区不会因写满导致生产行为被阻塞。



### 数据可靠性

Kafka 提供了三种可靠性级别，可以根据对可靠性和延迟的要求进行权衡配置。

- 0：producer 不等待 broker 的 ack，这一操作提供了一个最低的延迟。broker 一接收到数据还没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据；
- 1：producer 等待 broker 的 ack，partition 中的 leader 落盘成功后返回 ack。如果在 follower 同步成功之前 leader 故障，那么将会丢失数据；
- -1（all）：producer 等待 broker 的 ack，partition 中的 leader 和 follower 全部落盘成功后才返回 ack。

虽然 -1 级别在最大程度上保证数据不会丢失，但在极端情况下，当 ISR 中只有 leader 时，此时等同于级别 1，也可能会造成数据的丢失。如果在 follower 同步完成后，broker 发送 ack 之前，leader 发生了故障，那么会造成数据重复。



#### ISR

当级别为 -1 时，若某个 follower 因为故障迟迟不能与 leader 进行同步，那 leader 就要一直等下去，直到同步完成才能发送 ack。

为了解决这个问题，leader 维护了一个动态的 in-sync replica set（ISR），意为和 leader 保持同步的 follower 集合。当 ISR 中的 follower 完成数据的同步之后，leader 就会发送 ack。如果 follower 长时间未向 leader 发送通讯请求或同步数据，则该 follower 将被移出 ISR，该时间阈值由 replica.lag.time.max.ms 参数设定。当 leader 发生故障之后，就会从 ISR 中选举新的 leader。

注：0.9 版本之前，follower 加入 ISR 由两个条件决定：同步时间和数据相差的条数，0.9 版本之后只由时间决定。原因：生产者往往是批量插入数据的，当插入的数据量大于设置的 ISR 相差条数时，所有的 follower 都会从 ISR 中移除，待同步完成后又加入到 ISR 中。但后续的批量插入又会造成这种现象的循环，这样会频繁操作内存，增加性能消耗。



#### Exactly Once 语义

- At Least Once：将 ACK 级别设置为 -1，可以保证 producer 到 server 之间不会丢失数据。可以保证数据不丢失，但是不能保证数据不重复。
- At Most Once：将 ACK 级别设置为 0，可以保证生产者的每条消息只会发送一次。可以保证数据不重复，但是不能保证数据不丢失。
- Exactly Once：对于一些非常重要的信息，要求数据既不重复也不丢失。

在 0.11 版本以前的 Kafka，对 Exactly Once 是无能为力的。只能保证数据不丢失，在消费者端对数据做全局去重。对于多个下游应用的情况，每个都需要单独做全局去重，这就对性能造成了很大影响。



##### 幂等性

0.11 版本的 Kafka 引入了一项重大特性：幂等性。所谓的幂等性就是指 producer 不论向 broker 发送多少次重复数据，都只会持久化一条。幂等性结合 At Least Once 语义，就构成了 Exactly Once 语义。要启用幂等性，只需要将 producer 的参数中 enable.idompotence 设置为 true 即可。

开启幂等性的 producer 在初始化的时候会被分配一个 PID，发往同一 partition 的消息会附带 sequence number（单调自增）。broker 端会对 <PID, Partition, SeqNumber> 做缓存，当具有相同主键的消息提交时，broker 只会持久化一条。

但是 PID 在 producer 重启后就会变化，同时不同的 partition 也具有不同主键，所以**幂等性无法保证跨分区跨会话的 Exactly Once**。

