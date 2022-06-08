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

