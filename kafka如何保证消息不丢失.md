## 前言

kafka对于消息的可靠性保证。作为消息组件，保证消息不丢失，是非常重要的。

那么kafka是如何保证消息不丢失的呢？

---------------------------------------

## 前提条件

任何消息组件不丢数据都是在特定场景下一定条件的，`kafka`要保证消息不丢，有两个核心条件。

### 第一.必须是已提交的消息即committed message

`kafka`对于`committed message`的定义是，生产者提交消息到`broker`，并等到多个`broker`确认并返回给生产者已提交的确认信息。而这`多个broker`是由我们自己来定义的，可以选择只要有一个`broker`成功保存该消息就算是已提交，也可以是令所有`broker`都成功保存该消息才算是已提交。不论哪种情况，`kafka`只对已提交的消息做持久化保证。

###第二.N 个`broker`中至少有 1 个存活

虽然`kafka`集群是分布式的，但也必须保证有足够`broker`正常工作，才能对消息做持久化做保证。也就是说 `kafka`不丢消息是有前提条件的，假如你的消息保存在 N 个`kafka broker`上，那么这个前提条件就是这 N 个`broker`中至少有 1 个存活。只要此条件成立，`kafka`就能保证你的这条消息永远不会丢失。

##如何保证消息不丢

一条消息从产生，到发送到`kafka`保存，到被取出消费，会有多个场景和流程阶段，很可能会出现丢失情况，上文描述了消息丢失的几种情况，下面简单介绍下如何保证消息不丢失。

### 生产端

`Producer`端可能会丢失消息。目前`Kafka Producer`是异步发送消息的，也就是说如果你调用的是`producer.send(msg)`这个`API`，那么它通常会立即返回，但此时你不保证消息发送已成功完成。可能会出现：网络抖动，导致消息压根就没有发送到`Broker`端；或者消息本身不合规导致`Broker`拒绝接收（比如消息太大了，超过了`Broker`的限制）。

使用`producer.send(msg, callback)`接口就能避免这个问题，根据回调，一旦出现消息提交失败的情况，就可以有针对性地进行处理。如果是因为那些瞬时错误，`Producer`重试就可以了；如果是消息不合规造成的，那么调整消息格式后再次发送。总之，处理发送失败的责任在`Producer`端而非`Broker`端。当然，如果此时`broker`宕机，那就另当别论。

####1.send(msg, callback)

可以从`callback`回调中得到该条消息的发送结果，并且`callback`是异步回调，所以在兼具性能的情况下，也对消息具有比较好的掌控。

```java
 ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", key, value);
 producer.send(myRecord,
           new Callback() {
               public void onCompletion(RecordMetadata metadata, Exception e) {
                   if(e != null) {
                      e.printStackTrace();
                   } else {
                      System.out.println("The offset of the record we just sent is: " + metadata.offset());
                   }
               }
           });
```

当我们通过  `send(msg, callback)` 是不是就意味着消息一定不丢失了呢？
答案明显是：不是!

我们接着上面，`send(msg, callback)`里面 `callback`返回的成功，

到底是不是真的确保消息万无一失了？答案是否定的！

####2.request.required.acks 参数

还需要通过 `request.required.acks `参数来设置数据可靠性的级别！

```java
 props.put("acks", "all");//或者-1
```

- 1（默认）：这意味着 producer 在 `ISR副本同步队列`中的 leader 已成功收到的数据并得到确认后发送下一条 message。如果 leader 宕机了，则会丢失数据。
- 0：这意味着 producer 无需等待来自 broker 的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
- -1：producer 需要等待 ISR 中的所有 follower都确认接收到数据后才算一次发送完成，可靠性最高。**但是这样也不能保证数据不丢失**，比如当 ISR 中只有 leader时（ISR 中的成员由于某些情况会增加也会减少，最少就只剩一个 leader），这样就变成了 acks=1 的情况。

如果要提高数据的可靠性，在设置 `request.required.acks=-1` 的同时，也要设置最小写入副本数的个数`min.insync.replicas` 参数 (可以在` broker `或者 `topic` 层面进行设置) 的配合，这样才能发挥最大的功效。

`min.insync.replicas` 这个参数设定 `ISR` 中的最小副本数是多少，默认值为 1，当且仅当 `request.required.acks` 参数设置为 -1 时，此参数才生效。

如果 ISR 中的副本数少于 `min.insync.replicas` 配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。

### Broker 端的配置

其实到这里，生产者端基本已经做好了数据不丢失的大部分准备，但是有些东西是要配合Broker 端一起，才能达到预期的不丢失数据的， 比如上面说到的

- `min.insync.replicas` 配置
  我们上面知道，当 生产者 `acks = -1` 的时候，写入的副本数就必须 >= `min.insync.replicas` 数，
  当达不到这个要求的时候，生产者端会收到一个`either NotEnoughReplicas or NotEnoughReplicasAfterAppend`的异常。

  所以我们这个`min.insync.replicas`参数必须不能大于数据冗余备份数 `replication.factor` 。否则生产者将无法写入任何数据，一般建议 `replication.factor` 数要大于 `min.insync.replicas`，比如3个机器的集群，设置 `replication.factor` = 3，那么设置 `min.insync.replicas` = 2 即可，这样既保证了数据write的时候有一个副本的冗余，也能保证在一些情况下，某台Broker宕机导致数据无法达到3个副本时，依然可以正常write数据。

- `unclean.leader.election.enable`
  这里 Broker 端还有一个重要的配置就是 `unclean.leader.election.enable = false`
  这个配置代表着一些数据落后比较多的 follower，是否能在leader宕机后被选举成新的 leader，如果你设置成 true，很明显，如果这样的follower成为新leader，就会造成最新的一部分数据丢失掉，

### 消费端

`Consumer`端丢数据的情况稍微复杂些。`Consumer`”位移“(`offset`)，表示`Consumer`当前消费到`topic`分区的哪个位置。

`kafka`通过先消费消息，后更新`offset`，来保证消息不丢失。但是这样可能会出现消息重复的情况，具体如何保证`only-once`,前文已提到过。

当我们`consumer`端开启多线程异步去消费时，情况又会变得复杂一些。此时`consumer`自动地向前更新`offset`，假如其中某个线程运行失败了，它负责的消息没有被成功处理，但位移已经被更新了，因此这条消息对于`consumer`而言实际上是丢失了。

这里的关键就在自动提交`offset`，如何真正地确认消息是否真的被消费，再进行更新`offset`。此问题的解决方式：如果是多线程异步处理消费消息，`consumer`不要开启自动提交`offset`，`consumer`端程序自己来处理`offset`的提交更新。



## 实践配置

最后分享下`kafka`无消息丢失配置：

- `producer`端使用`producer.send(msg, callback)`带有回调的`send`方法。
- 设置`acks = all`。`acks`是`Producer`的一个参数，代表“已提交”消息的定义。如果设置成`all`，则表明所有`Broker`都要接收到消息，该消息才算是“已提交”。
- 设置`retries`为一个较大的值。同样是`Producer`的参数。当出现网络抖动时，消息发送可能会失败，此时配置了`retries`的`Producer`能够自动重试发送消息，尽量避免消息丢失。
- 设置`unclean.leader.election.enable = false`。这是`Broker`端的参数，在`kafka`版本迭代中社区也多次反复修改过他的默认值，之前比较具有争议。它控制哪些`Broker`有资格竞选分区的`Leader`。如果一个`Broker`落后原先的`Leader`太多，那么它一旦成为新的`Leader`，将会导致消息丢失。故一般都要将该参数设置成`false`！！！
- 设置`replication.factor >= 3`。这也是`Broker`端的参数。保存多份消息冗余。
- 设置`min.insync.replicas > 1`。`Broker`端参数，控制消息至少要被写入到多少（一个以上）个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在生产环境中不要使用默认值 1 ！！！确保`replication.factor > min.insync.replicas`。如果两者相等，那么只要有一个副本离线，整个分区就无法正常工作了。推荐设置成`replication.factor = min.insync.replicas + 1`。
- 确保消息消费完成再提交。`Consumer`端有个参数`enable.auto.commit`，最好设置成`false`，并自己来处理`offset`的提交更新。



------------------------







参考文献

- [Apache Kafka](https://kafka.apache.org/documentation/#clientconfig)
- [kafka 如何解决消息队列重复消费 (zhouyuxin.net)](https://www.zhouyuxin.net/article/137)

- [Kafka数据可靠性深度解读-InfoQ](https://www.infoq.cn/article/depth-interpretation-of-kafka-data-reliability)
- [kafka是如何保证消息不丢失的 - 云+社区 - 腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1589157)
- [Kafka保证消息不丢失不重复 - 简书 (jianshu.com)](https://www.jianshu.com/p/4e6f01b4259d)
- [Kafka ——如何保证消息不会丢失 - 简书 (jianshu.com)](https://www.jianshu.com/p/68c173e4c549)
- [kafka如何保证消息队列不丢失? - gaopengpy - 博客园 (cnblogs.com)](https://www.cnblogs.com/gaopengpy/p/13516216.html)

