# Kafka: a distributed messaging system for log processing

### abstract

​        日志处理已成为消费者互联网公司数据通信的关键。Kafka是一个分布式消息传递系统，用于收集和提供低迟

的大量日志数据，且融合了现有日志聚合器和消息系统的设计思想，让此系统适用于离线和在线消息消费。我们在

Kafka中做了很多非常规但实用的设计选择，以使系统高效和可扩展。实验结果表明，与现有的两种消息系统相

比，Kafka具有卓越的性能。Kafka投入生产已经一段时间了，平均每天处理数百GB的新数据。



####General Terms

协作管理，性能，设计思想，实验



#### key words

消息传递，分布式，日志处理，吞吐量，在线系统。



### 1, 简介

​		大型互联网公司会产生大量的“日志”数据。该数据通常包括：（1）登录，浏览量，点击，共享，评论和搜索

查询等用户活动事件; （2）运营指标，例如服务调用，时延，及每台计算机上的CPU，内存，网络或磁盘利用率

等。日志数据用于跟踪用户行为。

​		我们为日志处理构建了一个新颖的消息系统Kafka，它结合了传统日志聚合器和消息系统的优点。一方面，

Kafka是分布式的，易于扩展，高吞吐量。另一方面，Kafka提供类似于消息传递系统的API，并允许应用程序实时

访问日志。 Kafka已经开源并在LinkedIn上成功使用了6个多月，它极大地简化了我们的基础架构，我们可以利用

单个软件，在线或者离线消费所有类型的日志数据。本文的其余部分安排如下，第2节中分析传统的消息传递系统

和日志聚合器。第3节中，描述Kafka的体系结构及其关键部分的设计原则，第4节讲述Kafka在LinkedIn上的应

用，第5节展示Kafka的性能结果，最后讨论了未来的工作并在第6节中得出结论。



### 2, 相关的工作

​      现在企业级消息消费系统存在以下问题：

1. 企业系统关注于数据量来源丰富的分支，这样做会损失日志数据
2. 放松了对这些软件初始设计约束，反而过分关注吞吐量
3. weak in distributed support
4. 企业系统总是认为它们能做到接近即时消费，这样导致未消费队列总是太小。



### 3, Kafka体系结构与设计总则

​      由于现有系统的局限，我们设计出了一个基于消息消费的日志聚合Kafka。

 producer：可以在一次生产请求中发送一批消息。

consumer：为了订阅topic，consumer首先对topic创建了大于等于一个的message streams，这样，消息到达

​					  topic的时候，就会平均分配给这些sub-streams(Kafka如何分配这些消息在3.2节介绍)。在消息被创

​					  建的时候，每个message stream会为持续的消息流中每一条消息提供一个iterator interface。之

​				      后，consumer会遍历这些消息，从不停止，如果没有消息了，就会阻塞直到有新的消息到达topic。

在consumer group中Kafka使用两种消息模型，即点对点模型(P2P)和发布订阅(pub/sub)模型。

1. p2p模型：多个consumer协作消费一个topic中的消息副本
2. publish／subscribe模型：多个consumer消费订阅的topic中属于自己的副本

 ![image-20190715212806997](/Users/didi/Library/Application Support/typora-user-images/image-20190715212806997.png)

​                                                                        图1，Kafka架构



#### 3.1 一个分区的效率

​		几个让系统更高效的策略。

##### （1）简单的存储

​		每一个topic的partition对应一个logical log，物理上讲，一个log会被分为大小相同的几个segment file。

producer产生的消息放入partition，即是被broker追加到最后一个segment file，为了更好的性能，我们仅在已

发布可配置数量的消息或经过一定时间后才将段文件刷新到磁盘，最后再展示给consumer.

​		不同于典型的消息消费系统，存在Kafka的消息没有明确的消息id，取而代之的是，每条消息被所属log中的

logical offset标记，当然，我们会为消息标记 增加的但非连续的id,为了计算下一条消息的id，要用当前消息的长

度加id来计算。

​		consumer总是按序消费来自特定分区的消息，如果消费者获知了一个特定的消息偏移，则意味着在这个偏移

量之前的消息均被消费者消费过了。消费者向broker发出异步pull请求，以便为应用程序准备好数据缓冲区。每个

pull请求包含消费起始的offset和可接收的字节数，每个broker在内存中均会保存有序的offsets list以及每个

segment file中第一条消息的偏移量。Broker会通过offset list定位segment file，并将数据发送回消费者，当消费

者接受到消息后，会计算下一个要消费的消息的偏移量，并在下一个请求中使用此偏移量。 Kafka日志的布局和内

存中的索引如图2所示。每个框都显示了消息的偏移量。

![image-20190716170148660](/Users/didi/Library/Application Support/typora-user-images/image-20190716170148660.png)

 ##### (2) 有效的传输

​		之前提到producer会一次产生很多消息，所以consumer会在一次请求中检索非常多的消息，与传统方式不

同的是，使用页缓存，会避免double-buffering，优点在于当broker进程重启的时候，warm cache依然可以保

留，因为Kafka不缓存正在处理中的message，所以不会占用太多的内存。

​		Producer与consumer，有序访问segment files，consumer会比producer滞后一些，这对于连续写入与预

读而言的操作而言是高效的。

​		另外，我们还优化了consumer的访问过程，Kafka是一个多订阅者的系统，一条消息可能会被不同的

consumer所在的应用消费多次，一个本地消息文件发给端上的方式有如下几个过程：

​		1). 将缓存媒介的数据读入系统的页缓存中

​		2). 将页缓存中的数据的复本读入到应用的缓存中

​		3). 把应用的缓存读入到另一个kernel的缓存中

​		4). 把kernel的缓存发给socket

​		可以看到以上四个过程包含四次复制和两次系统调用，我们使用了sendfile API后就可以减少两次复制与一次

系统调用，所以，应用sendfile API提高了效率。

      ##### （3）无状态传输

 		在consumer与broker的交互过程中，broker是不记录consumer消费的消息数量的，这些消息消费数量由

consumer自己记录，这样的设计可以降低broker的复杂度，但是，这样也意味着broker无法知道是否所有的订阅

者是否已经消费了当前之前的消息，所以，Kafka采用了一个基于时间的SLA算法，七天内自动清除消息。

 		这个设计还有一个附带的好处，让consumer可以随意的转向去取old offset，并达到再次消费。比如例子1：

当consumer端的应用日志出现问题时，应用可以在日志修复之后，再去消费这些消息。例子2：消费后的数据有

时是会被永久存储的，如果此时consumer挂掉，未来得及存盘的消息就会丢失，这样的情况下，当consumer重

启之后，可以检查最小的offset，这样的模式对于请求模型来说要优于推送模型。



#### 3.2 分布式协调

​		我们的目标是将存储在broker中的消息平均地分配给consumer，同时不会引入太多的协作问题。

​		第一种方式是将一个topic平均地分区，这表示，在任何一个给定的时间，一个partition只能被一个

consumer组中的一个consumer消费，假定我们允许多个consumer同时消费一个partition，那么它们就会相互

协调到底哪位consumer消费哪些消息，这就引入了锁机制和状态保持机制。相比之下，在我们的设计中，消息消

费进程这一协作过程仅需要在负载均衡时触发时启动。因此，为了达到真正的负载均衡，需要做到partition的数

量大于consumer组中的consumer数量。

​        第二种方式是，放弃使用‘master’模式，让consumer使用一种权利下降的协调方式，因为这样可以避免

‘master’失效带来的一些列问题，即是使用zookeeper，ZOOKEEPER有着简单的，类似于sendfile API的文件系

统。Kafka使用zookeeper做了以下事情：

1. 监听消息通道，增加或者移除brokers与consumers

2. 当一些事件发生时，触发consumer的重平衡机制

3. 维持consumer与broker的消费关系，并且记录consumer在其partition的已经消费过的offset。

   

​        综合来讲，Zookeeper就是在做一些监听，登记注册的事情，每个使用者组都与Zookeeper中的所有权限册

表和偏移注册表相关联。所有权注册表对于每个订阅的分区都有一个路径，路径值是当前从该分区消耗的

consumer的ID（我们使用consumer拥有该分区的术语）。偏移注册表存储每个订阅分区，即分区中最后消耗的

消息的偏移量。

​		在Zookeeper中创建的通道对于broker注册表，使用者注册表和所有权注册表来说是短暂的，但是对于偏移

注册表是持久的。如果broker挂掉，则broker上的所有分区都将自动从broker注册表中删除。consumer一旦实

效，那么它在consumer注册表中的条目以及它在所有权注册表中拥有的所有分区都将被清除。每个消费者在

broker注册表和使用者注册表上都会注册Zookeeper观察程序，只要broker集成或消费者组发生更改，观察程序

就会发出通知。

​		在consumer启动初始，或当consumer得知broker/consumer有变动时，consumer启动重新平衡过程以确

定它应该消费的新分区子集。该过程在算法1中描述。

![image-20190716112718482](/Users/didi/Library/Application Support/typora-user-images/image-20190716112718482.png)

​																				算法1，重平衡过程

​		当组内有多个consumer时，每个consumer都会收到broker/consumer变更的通知。但是，consumer的通

知可能会略有不同。因此，consumer-A可能试图取得仍由consumer-B拥有的分区的所有权。发生这种情况时，

consumer-B只是释放它当前拥有的所有分区，等待一会并重试重新平衡过程。实际上，重新平衡过程通常在重试

几次后就会稳定下来。

​		创建新的使用者组时，偏移注册表中没有可用的偏移量。在这种情况下，使用我们在broker上提供的API，

consumer将从每个订阅分区上可用的最小或最大偏移（取决于配置）开始。



#### 3.3 交付保证

​		通常Kafka只保证at-least-delivery，因为exactly-once-delivery需要两步交付，对应用来说是非必需的。大多

数时候，一条消息给每个consumer组恰好提交一次，然而，在一种情况下，当某个consumer进程崩溃没有彻底

关停时，其他的consumer进程会从挂掉的consumer所拥有的分区中拿到一些已经成功提交给Zookeeper的副

本，如果一些应用对这些副本较为敏感的话，就会给这些副本加上自己的标志，并且给consumer返回这些消息的

独特标志，这样的话，就比两步提交还要低效。

​        Kafka保证交付给consumer的来自一个分区消息都是有序的，但是不保证来自不同分区的消息被交付给

consumer是有序的。

​        为了避免log corruption，Kafka为日志中的每条消息都存储了一个CRC检验，如果broker上有任何I/O错

误，Kafka就会启动一个恢复进程来移除那些与CRC码不一致的消息。在消息层面上加上CRC验证还可以帮助我们

检验消息被生产之后或被消费之后的网络错误。



### 4, Kafka在Linked上的应用

 		下图3展示了我们部署的简单模型，在线datacenter中，前端服务生产出各种各样的消息并以批处理的方式发

给本地的Kafka brokers，我们使用负载平衡器将发布请求平均地发给brokers，consumers运行在同一个

datacenter中。

![image-20190716163004288](/Users/didi/Library/Application Support/typora-user-images/image-20190716163004288.png)

​        同样地，我们也部署了一个离线的分析datacenter，并靠近于DWH 与Hadoop等设施。离线的Kafka中内

嵌的consumer从在线的datacenter请求数据，之后将数据的副本传给DWH 和Hadoop去完成一些数据分析与处

理的工作。

​        为了使数据不丢失，为每条生产出来的消息带上时间戳和server name，我们也会定期让producer产生一

个监测事件，这样在固定的时间窗内，可以记录producer产生的所有消息数，同样的，consumer也可以根据数量

来确定它从partition中消费的消息数量，并让监测事件可以确定消息的正确性。



### 5, 实验结果

producer端测试

​		我们设置了一个broker，进行异步消息存盘，对于每个系统，让一个producer发一千万条消息。X轴代表

发送给broker的消息数量(以MB为单位)，y轴表示producer每秒产生的消息数。为什么Kafka表现的更佳呢？

1. Kafkaproducer不等待broker通知，所以producer发送消息可以做到与broker接收消息一样快。当然这样做

   也有问题，无通知就无法保证消息全部到达了broker，这对于对交易消息要求严格的应用来说不是好事。

2. Kafka有高效的存储模式，它将请求头限定在9字节大小，而ActiveMQ是144字节，会比Kafka多占用70%的空

   间。实验证明，ActiveMQ主要花费时间在遍历B-树上。

3. 最后，通过分摊RPC开销，批处理大大提高了吞吐量。 在Kafka中，批量大小为50的消息将吞吐量提高了近一

   个数量级。 

  

consumer端测试，Kafka表现优秀的原因

		1. Kafka有高效的存储模式。
  		2. activeMQ与rabbitMQ要维持传输状态。
  		3. 使用了sendfileAPI,减少了传输时延。



 ### 结论

 		未来要优化的部分

1. 加内置的replica，处理不可恢复的错误。
2. 增加流处理能力





