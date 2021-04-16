---
title: RocketMQ-03-保证消息可靠性
copyright: true
comments: true
related_posts: true
date: 2021-04-12 23:24:23
tags: 保证消息可靠性
categories: RocketMQ
---

# Rocketmq如何保证消息不丢失，如何保证消息不被重复消费

转载至：作者：Java余笙
链接：https://www.jianshu.com/p/cb414cf3f098


## 1、消息整体处理过程

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMjQyMTgyOS01ODhiNjNiYTM5ZTc3YjQ2LnBuZw?x-oss-process=image/format,png)

这里我们将消息的整体处理阶段分为3个阶段进行分析：

**Producer发送消息阶段。**

**Broker处理消息阶段。**

**Consumer消费消息阶段。**

### Producer发送消息阶段

发送消息阶段涉及到Producer到broker的网络通信，因此丢失消息的几率一定会有，那RocketMQ在此阶段用了哪些手段保证消息不丢失了（或者说降低丢失的可能性）。

#### 手段一：提供SYNC的发送消息方式，等待broker处理结果。

RocketMQ提供了3种发送消息方式，分别是：

- 同步发送：Producer 向 broker 发送消息，阻塞当前线程等待 broker 响应 发送结果。
- 异步发送：Producer 首先构建一个向 broker 发送消息的任务，把该任务提交给线程池，等执行完该任务时，回调用户自定义的回调函数，执行处理结果。
- Oneway发送：Oneway 方式只负责发送请求，不等待应答，Producer只负责把请求发出去，而不处理响应结果。

我们在调用producer.send方法时，不指定回调方法，则默认采用同步发送消息的方式，这也是丢失几率最小的一种发送方式。

#### 手段二：发送消息如果失败或者超时，则重新发送。

- 发送重试源码如下，本质其实就是一个for循环，当发送消息发生异常的时候重新循环发送。默认重试3次，重试次数可以通过producer指定。

#### 手段三：broker提供多master模式，即使某台broker宕机了，保证消息可以投递到另外一台正常的broker上。

- 如果broker只有一个节点，则broker宕机了，即使producer有重试机制，也没用，因此利用多主模式，当某台broker宕机了，换一台broker进行投递。

总结

- producer消息发送方式虽然有3种，但为了减小丢失消息的可能性尽量采用同步的发送方式，同步等待发送结果，利用**同步发送+重试机制+多个master节点**，尽可能减小消息丢失的可能性。

### Broker处理消息阶段

#### 手段四：提供同步刷盘的策略

public enum FlushDiskType { SYNC_FLUSH, //同步刷盘 ASYNC_FLUSH//异步刷盘（默认） }

我们知道，当消息投递到broker之后，会先存到page cache，然后根据broker设置的刷盘策略是否立即刷盘，也就是如果刷盘策略为异步，broker并不会等待消息落盘就会返回producer成功，也就是说当broker所在的服务器突然宕机，则会丢失部分页的消息。

#### **手段五：提供主从模式，同时主从支持同步双写**

即使broker设置了同步刷盘，如果主broker磁盘损坏，也是会导致消息丢失。 因此可以给broker指定slave，同时设置master为SYNC_MASTER，然后将slave设置为同步刷盘策略。

此模式下，producer每发送一条消息，都会等消息投递到master和slave都落盘成功了，broker才会当作消息投递成功，保证休息不丢失。

总结

在broker端，消息丢失的可能性主要在于刷盘策略和同步机制。
RocketMQ默认broker的刷盘策略为异步刷盘，如果有主从，同步策略也默认的是异步同步，这样子可以提高broker处理消息的效率，但是会有丢失的可能性。因此可以通过同步刷盘策略+同步slave策略+主从的方式解决丢失消息的可能。

### Consumer消费消息阶段

手段六：consumer默认提供的是At least Once机制

从producer投递消息到broker，即使前面这些过程保证了消息正常持久化，但如果consumer消费消息没有消费到也不能理解为消息绝对的可靠。因此RockerMQ默认提供了At least Once机制保证消息可靠消费。

**何为At least Once？**

Consumer先pull 消息到本地，消费完成后，才向服务器返回ack。

通常消费消息的ack机制一般分为两种思路：

1、先提交后消费；

2、先消费，消费成功后再提交；

思路一可以解决重复消费的问题但是会丢失消息，因此Rocketmq默认实现的是思路二，由各自consumer业务方保证幂等来解决重复消费问题。

#### 手段七：消费消息重试机制

当消费消息失败了，如果不提供重试消息的能力，则也不能算完全的可靠消费，因此RocketMQ本身提供了重新消费消息的能力。

总结

consumer端要保证消费消息的可靠性，主要通过At least Once+消费重试机制保证。

## 2、如何保证消息不被重复消费

回答这个问题，首先你别听到重复消息这个事儿，就一无所知吧，**你先大概说一说可能会有哪些重复消费的问题。**

首先，比如 RabbitMQ、RocketMQ、Kafka，都有可能会出现消息重复消费的问题，正常。因为这问题通常不是 MQ 自己保证的，是由我们开发来保证的。挑一个 Kafka 来举个例子，说说怎么重复消费吧。

Kafka 实际上有个 offset 的概念，就是每个消息写进去，都有一个 offset，代表消息的序号，然后 consumer 消费了数据之后，每隔一段时间（定时定期），会把自己消费过的消息的 offset 提交一下，表示“我已经消费过了，下次我要是重启啥的，你就让我继续从上次消费到的 offset 来继续消费吧”。

但是凡事总有意外，比如我们之前生产经常遇到的，就是你有时候重启系统，看你怎么重启了，如果碰到点着急的，直接 kill 进程了，再重启。这会导致 consumer 有些消息处理了，但是没来得及提交 offset，尴尬了。重启之后，少数消息会再次消费一次。

有这么个场景。数据 1/2/3 依次进入 kafka，kafka 会给这三条数据每条分配一个 offset，代表这条数据的序号，我们就假设分配的 offset 依次是 152/153/154。消费者从 kafka 去消费的时候，也是按照这个顺序去消费。假如当消费者消费了 offset=153 的这条数据，刚准备去提交 offset 到 zookeeper，此时消费者进程被重启了。那么此时消费过的数据 1/2 的 offset 并没有提交，kafka 也就不知道你已经消费了 offset=153 这条数据。那么重启之后，消费者会找 kafka 说，嘿，哥儿们，你给我接着把上次我消费到的那个地方后面的数据继续给我传递过来。由于之前的 offset 没有提交成功，那么数据 1/2 会再次传过来，如果此时消费者没有去重的话，那么就会导致重复消费。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMjQyMTgyOS04YzJjZDA0NGYzOTIzMDE1LnBuZw?x-oss-process=image/format,png)

 

如果消费者干的事儿是拿一条数据就往数据库里写一条，会导致说，你可能就把数据 1/2 在数据库里插入了 2 次，那么数据就错啦。

其实重复消费不可怕，可怕的是你没考虑到重复消费之后，怎么保证**幂等性**。

举个例子吧。假设你有个系统，消费一条消息就往数据库里插入一条数据，要是你一个消息重复两次，你不就插入了两条，这数据不就错了？但是你要是消费到第二次的时候，自己判断一下是否已经消费过了，若是就直接扔了，这样不就保留了一条数据，从而保证了数据的正确性。

一条数据重复出现两次，数据库里就只有一条数据，这就保证了系统的幂等性。

幂等性，通俗点说，就一个数据，或者一个请求，给你重复来多次，你得确保对应的数据是不会改变的，不能出错。

### 所以第二个问题来了，怎么保证消息队列消费的幂等性？

其实还是得结合业务来思考，我这里给几个思路：

比如你拿个数据要写库，你先根据主键查一下，如果这数据都有了，你就别插入了，update 一下好吧。

比如你是写 Redis，那没问题了，反正每次都是 set，天然幂等性。

比如你不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据的时候，里面加一个全局唯一的 id，类似订单 id 之类的东西，然后你这里消费到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个 id 写 Redis。如果消费过了，那你就别处理了，保证别重复处理相同的消息即可。

比如基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复数据插入只会报错，不会导致数据库中出现脏数据。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMjQyMTgyOS03NGNiNDg4ZTlmMzMzMGRmLnBuZw?x-oss-process=image/format,png)