---
title: RocketMQ-02-RocketMQ快速入门
copyright: true
related_posts: true
date: 2021-01-06 20:59:26
tags: RocketMQ入门
categories: RocketMQ
---

# RocketMQ入门

## 核心概念说明

![image-20210106210340592](uploads/RocketMQ/RocketMQ01.png)

### Producer

- 消息生产者，负责产生消息，一般由业务系统负责产生消息。

#### Producer Group

- 一类 Producer 的集合名称，这类 Producer 通常发送一类消息，且发送逻辑一致。

### Consumer

- 消息费者，负责消费消息，一般是后台系统负责异步消费。

#### Push Consumer

- 服务端向消费者端推送消息

####   Pull Consumer

- 消费者端向服务定时拉取消息

####   Consumer Group

-  一类 Consumer 的集合名称，这类 Consumer 通常消费一类消息，且消费逻辑一致。

### NameServer

- 集群架构中的组织协调员

- 收集broker的工作情况
- 不负责消息的处理

###  Broker

- 是RocketMQ的核心负责消息的发送、接收、高可用等（真正干活的）
- 需要定时发送自身情况到NameServer，默认10秒发送一次，超时2分钟会认为该broker失效。

### Topic

- 不同类型的消息以不同的Topic名称进行区分，如User、Order等

- 是逻辑概念

### Message Queue

- 消息队列，用于存储消息



## 快速入门

### 创建topic

```java

import org.apache.rocketmq.client.producer.DefaultMQProducer;

public class TopicDemo {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("zlm-mqProducer");

        //设置nameserver的地址
        producer.setNamesrvAddr("192.168.62.90:9876");

        // 启动生产者
        producer.start();

        /**
         * 创建topic，参数分别是：broker的名称，topic的名称，queue的数量
         *
         */
        producer.createTopic("broker-zhangliming", "zhangliming-test-topic", 8);

        System.out.println("topic创建成功!");

        producer.shutdown();
    }
}

```

### 发送消息（同步）

```java
import org.apache.rocketmq.common.message.Message;

public class SyncProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("haoke");
        producer.setNamesrvAddr("192.168.62.90:9876");
        producer.start();
        //发送消息
        String msg = "我的第一个消息-add!";
        Message message = new Message("my-topic", "add", msg.getBytes("UTF-8"));
        SendResult sendResult = producer.send(message);
        System.out.println("消息id：" + sendResult.getMsgId());
        System.out.println("消息队列：" + sendResult.getMessageQueue());
        System.out.println("消息offset值：" + sendResult.getQueueOffset());
        System.out.println(sendResult);

        producer.shutdown();
    }
}


消息id：C0A800693F5814DAD5DC1E4E22F20000
消息队列：MessageQueue [topic=my-topic, brokerName=broker-zhangliming, queueId=3]
消息offset值：0
SendResult [sendStatus=SEND_OK, msgId=C0A800693F5814DAD5DC1E4E22F20000, offsetMsgId=C0A83E5A00002A9F00000000000000BA, messageQueue=MessageQueue [topic=my-topic, brokerName=broker-zhangliming, queueId=3], queueOffset=0]

```

### Message数据结构

| 字段名         | 默认值 | 说明                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| Topic          | null   | 必填，线下环境不需要申请，线上环境需要申请后才能使用         |
| Body           | null   | 必填，二进制形式，序列化由应用决定，Producer 与 Consumer 要协商好序列化形式。 |
| Tags           | null   | 选填，类似于 Gmail 为每封邮件设置的标签，方便服务器过滤使用。<br/>目前只支持每个消息设置一个 tag，所以也可以类比为 Notify 的 MessageType 概念 |
| Keys           | null   | 选填，代表这条消息的业务关键词，服务器会根据 keys 创建哈希索引，<br>设置后，可以在 Console 系统根据 Topic、Keys 来查询消息，<br>由于是哈希索引，请尽可能保证 key 唯一，例如订单号，商品 Id 等。 |
| Flag           | 0      | 选填，完全由应用来设置，RocketMQ 不做干预    |
| DelayTimeLevel | 0      | 选填，消息延时级别，0 表示不延时，大于 0 会延时特定的时间才会被消费 |
| WaitStoreMsgOK | True   | 选填，表示消息是否在服务器落盘后才返回应答。                 |

### 发送消息（异步）

```java

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

public class AsyncProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("haoke");
        producer.setNamesrvAddr("192.168.62.90:9876");
        producer.start();

        // 发送消息
        String msg = "我的第一个异步发送消息!";
        Message message = new Message("my-topic", msg.getBytes("UTF-8"));
        producer.send(message, new SendCallback() {
            public void onSuccess(SendResult sendResult) {
                System.out.println("发送成功了!" + sendResult);
                System.out.println("消息id：" + sendResult.getMsgId());
                System.out.println("消息队列：" + sendResult.getMessageQueue());
                System.out.println("消息offset值：" + sendResult.getQueueOffset());
            }
            public void onException(Throwable e) {
                System.out.println("消息发送失败!" + e);
            }
        });
//        producer.shutdown();
    }
}


发送成功了!SendResult [sendStatus=SEND_OK, msgId=C0A8006943E014DAD5DC1E5694380000, offsetMsgId=C0A83E5A00002A9F0000000000000174, messageQueue=MessageQueue [topic=my-topic, brokerName=broker-zhangliming, queueId=1], queueOffset=0]
消息id：C0A8006943E014DAD5DC1E5694380000
消息队列：MessageQueue [topic=my-topic, brokerName=broker-zhangliming, queueId=1]
消息offset值：0
```

### 消费消息

```java

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.common.protocol.heartbeat.MessageModel;

import java.io.UnsupportedEncodingException;
import java.util.List;

public class ConsumerDemo {

    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("haoke-consumer");
        consumer.setNamesrvAddr("192.168.62.90:9876");

        // 订阅消息，接收的是所有消息
//        consumer.subscribe("my-topic", "*");
        consumer.subscribe("my-topic", "add || update");

        consumer.setMessageModel(MessageModel.CLUSTERING);

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {

                try {
                    for (MessageExt msg : msgs) {
                        System.out.println("消息:" + new String(msg.getBody(), "UTF-8"));
                    }
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }

                System.out.println("接收到消息 -> " + msgs);

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 启动消费者
        consumer.start();
    }
}


消息:我的第一个消息-add!
消息:我的第一个消息-add!
接收到消息 -> [MessageExt [queueId=2, storeSize=186, queueOffset=0, sysFlag=0, bornTimestamp=1609938835416, bornHost=/192.168.62.1:8951, storeTimestamp=1609938835430, storeHost=/192.168.62.90:10911, msgId=C0A83E5A00002A9F0000000000000000, commitLogOffset=0, bodyCRC=2131316890, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='my-topic', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, CONSUME_START_TIME=1609939507601, UNIQ_KEY=C0A8006920E814DAD5DC1E4E1BD80000, WAIT=true, TAGS=add}, body=[-26, -120, -111, -25, -102, -124, -25, -84, -84, -28, -72, -128, -28, -72, -86, -26, -74, -120, -26, -127, -81, 45, 97, 100, 100, 33], transactionId='null'}]]
接收到消息 -> [MessageExt [queueId=3, storeSize=186, queueOffset=0, sysFlag=0, bornTimestamp=1609938837234, bornHost=/192.168.62.1:8964, storeTimestamp=1609938837241, storeHost=/192.168.62.90:10911, msgId=C0A83E5A00002A9F00000000000000BA, commitLogOffset=186, bodyCRC=2131316890, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='my-topic', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, CONSUME_START_TIME=1609939507601, UNIQ_KEY=C0A800693F5814DAD5DC1E4E22F20000, WAIT=true, TAGS=add}, body=[-26, -120, -111, -25, -102, -124, -25, -84, -84, -28, -72, -128, -28, -72, -86, -26, -74, -120, -26, -127, -81, 45, 97, 100, 100, 33], transactionId='null'}]]


```

## 消息过滤器

- RocketMQ支持根据用户自定义属性进行过滤，过滤表达式类似于SQL的where，如：a> 5 AND b ='abc'

- 原因是默认配置下，不支持自定义属性，需要设置开启 broker.conf中配置,见第一章可以看见配置
  enablePropertyFilter=true

```java

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.MessageSelector;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.message.MessageExt;

import java.io.UnsupportedEncodingException;
import java.util.List;

public class ConsumerFilter {

    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("haoke");
        consumer.setNamesrvAddr("192.168.62.90:9876");
        // 订阅消息，接收的是所有消息
//        consumer.subscribe("my-topic", "*");
        consumer.subscribe("my-topic-filter", MessageSelector.bySql("sex='女' AND age>=18"));

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {
                try {
                    for (MessageExt msg : msgs) {
                        System.out.println("消息:" + new String(msg.getBody(), "UTF-8"));
                    }
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }

                System.out.println("接收到消息 -> " + msgs);

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 启动消费者
        consumer.start();

    }
}

结果：
    消息:美女002
接收到消息 -> [MessageExt [queueId=2, storeSize=194, queueOffset=2, sysFlag=0, bornTimestamp=1609940231764, bornHost=/192.168.62.1:9804, storeTimestamp=1609940231769, storeHost=/192.168.62.90:10911, msgId=C0A83E5A00002A9F00000000000003CE, commitLogOffset=974, bodyCRC=1635276022, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='my-topic-filter', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=3, sex=女, CONSUME_START_TIME=1609940231776, UNIQ_KEY=C0A800692B3C14DAD5DC1E636A540000, WAIT=true, TAGS=delete, age=21}, body=[-25, -66, -114, -27, -91, -77, 48, 48, 50], transactionId='null'}]]
消息:美女002
接收到消息 -> [MessageExt [queueId=1, storeSize=194, queueOffset=0, sysFlag=0, bornTimestamp=1609940241237, bornHost=/192.168.62.1:9818, storeTimestamp=1609940241243, storeHost=/192.168.62.90:10911, msgId=C0A83E5A00002A9F0000000000000490, commitLogOffset=1168, bodyCRC=1635276022, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='my-topic-filter', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, sex=女, CONSUME_START_TIME=1609940241244, UNIQ_KEY=C0A8006908F014DAD5DC1E638F550000, WAIT=true, TAGS=delete, age=22}, body=[-25, -66, -114, -27, -91, -77, 48, 48, 50], transactionId='null'}]]
消息:美女002
接收到消息 -> [MessageExt [queueId=2, storeSize=194, queueOffset=3, sysFlag=0, bornTimestamp=1609940251652, bornHost=/192.168.62.1:9836, storeTimestamp=1609940251657, storeHost=/192.168.62.90:10911, msgId=C0A83E5A00002A9F0000000000000552, commitLogOffset=1362, bodyCRC=1635276022, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='my-topic-filter', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=4, sex=女, CONSUME_START_TIME=1609940251658, UNIQ_KEY=C0A8006934D414DAD5DC1E63B8030000, WAIT=true, TAGS=delete, age=23}, body=[-25, -66, -114, -27, -91, -77, 48, 48, 50], transactionId='null'}]]

    




import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;

public class SyncProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("haoke");
        producer.setNamesrvAddr("192.168.62.90:9876");
        producer.start();

        //发送消息
        String msg = "这是一个用户的消息, id = 1003";
        Message message = new Message("my-topic-filter", "delete", msg.getBytes("UTF-8"));
        message.putUserProperty("sex","女");
        message.putUserProperty("age","21");
        SendResult sendResult = producer.send(message);
        System.out.println("消息id：" + sendResult.getMsgId());
        System.out.println("消息队列：" + sendResult.getMessageQueue());
        System.out.println("消息offset值：" + sendResult.getQueueOffset());
        System.out.println(sendResult);

        producer.shutdown();
    }
}

结果：
    消息id：C0A80069251814DAD5DC1E63D08D0000
消息队列：MessageQueue [topic=my-topic-filter, brokerName=broker-zhangliming, queueId=0]
消息offset值：0
SendResult [sendStatus=SEND_OK, msgId=C0A80069251814DAD5DC1E63D08D0000, offsetMsgId=C0A83E5A00002A9F0000000000000614, messageQueue=MessageQueue [topic=my-topic-filter, brokerName=broker-zhangliming, queueId=0], queueOffset=0]

```

## producer详解

### 顺序消息

在某些业务中，consumer在消费消息时，是需要按照生产者发送消息的顺序进行消费的，比如在电商系统中，订单的消息，会有创建订单、订单支付、订单完成，如果消息的顺序发生改变，那么这样的消息就没有意义了。

![image-20210106210340592](uploads/RocketMQ/RocketMQ02.png)



```java

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

public class OrderProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("HAOKE_ORDER_PRODUCER");
        producer.setNamesrvAddr("192.168.62.90:9876");
        producer.start();
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10; // 模拟生成订单id
            String msgStr = "order --> " + i +", id = "+ orderId;
            Message message = new Message("haoke_order_topic", "ORDER_MSG",
                    msgStr.getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(message, (mqs, msg, arg) -> {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }, orderId);
            System.out.println(sendResult);
        }
        producer.shutdown();
    }

}
```

```java


import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeOrderlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerOrderly;
import org.apache.rocketmq.common.message.MessageExt;

import java.io.UnsupportedEncodingException;
import java.util.List;

public class OrderConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new
                DefaultMQPushConsumer("HAOKE_ORDER_CONSUMER");
        consumer.setNamesrvAddr("192.168.62.90:9876");
        consumer.subscribe("haoke_order_topic", "*");
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                for (MessageExt msg : msgs) {
                    try {
                        System.out.println(Thread.currentThread().getName() + " "
                                + msg.getQueueId() + " "
                                + new String(msg.getBody(),"UTF-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }

//                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        consumer.start();
    }
}
```

测试结果：相同订单id的消息会落到同一个queue中，一个消费者线程会顺序消费queue，从而实现顺序消费消
息。



## 分布式事务消息

### 回顾是什么事务

聊什么是事务，最经典的例子就是转账操作，用户A转账给用户B1000元的过程如下：
用户A发起转账请求，用户A账户减去1000元
用户B的账户增加1000元
如果，用户A账户减去1000元后，出现了故障（如网络故障），那么需要将该操作回滚，用户A账户增加1000元。
这就是事务。

### 分布式事务

随着项目越来越复杂，越来越服务化，就会导致系统间的事务问题，这个就是分布式事务问题。
分布式事务分类有这几种：

- 基于单个JVM，数据库分库分表了（跨多个数据库）。
- 基于多JVM，服务拆分了（不跨数据库）。
- 基于多JVM，服务拆分了 并且数据库分库分表了。

解决分布式事务问题的方案有很多，使用消息实现只是其中的一种。

### 原理

#### Half(Prepare) Message

指的是暂不能投递的消息，发送方已经将消息成功发送到了 MQ 服务端，但是服务端未收到生产者对该消息的二次
确认，此时该消息被标记成“暂不能投递”状态，处于该种状态下的消息即半消息。

#### Message Status Check

由于网络闪断、生产者应用重启等原因，导致某条事务消息的二次确认丢失，MQ 服务端通过扫描发现某条消息长
期处于“半消息”时，需要主动向消息生产者询问该消息的最终状态（Commit 或是 Rollback），该过程即消息回
查。



![image-20210106210340592](uploads/RocketMQ/RocketMQ03.png)

### 执行流程

![image-20210106210340592](uploads/RocketMQ/RocketMQ04.png)

1. 发送方向 MQ 服务端发送消息。
2. MQ Server 将消息持久化成功之后，向发送方 ACK 确认消息已经发送成功，此时消息为半消息。
3. 发送方开始执行本地事务逻辑。
4. 发送方根据本地事务执行结果向 MQ Server 提交二次确认（Commit 或是 Rollback），MQ Server 收到Commit 状态则将半消息标记为可投递，订阅方最终将收到该消息；MQ Server 收到 Rollback 状态则删除半消息，订阅方将不会接受该消息。
5. 在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达 MQ Server，经过固定时间后MQ Server 将对该消息发起消息回查。
6. 发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
7. 发送方根据检查得到的本地事务的最终状态再次提交二次确认，MQ Server 仍按照步骤4对半消息进行操作。



```java

import java.util.HashMap;
import java.util.Map;

public class TransactionListenerImpl implements TransactionListener {

    private static Map<String, LocalTransactionState> STATE_MAP = new HashMap<>();

    /**
     * 执行具体的业务逻辑
     *
     * @param msg 发送的消息对象
     * @param arg
     * @return
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            System.out.println("用户A账户减500元.");
            Thread.sleep(500); //模拟调用服务

//             System.out.println(1/0);
            System.out.println("用户B账户加500元.");
            Thread.sleep(800);
            STATE_MAP.put(msg.getTransactionId(), LocalTransactionState.COMMIT_MESSAGE);

            // 二次提交确认
//            return LocalTransactionState.UNKNOW;
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            e.printStackTrace();
        }

        STATE_MAP.put(msg.getTransactionId(), LocalTransactionState.ROLLBACK_MESSAGE);
        // 回滚
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }

    /**
     * 消息回查
     *
     * @param msg
     * @return
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.println("状态回查 ---> " + msg.getTransactionId() +" " +STATE_MAP.get(msg.getTransactionId()) );
        return STATE_MAP.get(msg.getTransactionId());
    }
}
```



```java

public class TransactionProducer {
    public static void main(String[] args) throws Exception {

        TransactionMQProducer producer = new TransactionMQProducer("transaction_producer");
        producer.setNamesrvAddr("192.168.62.90:9876");

        // 设置事务监听器
        producer.setTransactionListener(new TransactionListenerImpl());
        producer.start();

        // 发送消息
        Message message = new Message("pay_topic", "用户A给用户B转账500元".getBytes("UTF-8"));
        producer.sendMessageInTransaction(message, null);

        Thread.sleep(99999);
        producer.shutdown();
    }
}
```



```java

public class TransactionConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("HAOKE_CONSUMER");
        consumer.setNamesrvAddr("192.168.62.90:9876");
        // 订阅topic，接收此Topic下的所有消息
        consumer.subscribe("pay_topic", "*");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                            ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    try {
                        System.out.println(new String(msg.getBody(), "UTF-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
    }
}
```

#### 测试结果：

返回commit状态时，消费者能够接收到消息，返回rollback状态时，消费者接受不到消息。

## consumer详解

### push和pull模式

在RocketMQ中，消费者有两种模式，一种是push模式，另一种是pull模式。
- (常用)push模式：客户端与服务端建立连接后，当服务端有消息时，将消息推送到客户端。
- pull模式：客户端不断的轮询请求服务端，来获取新的消息。
但在具体实现时，Push和Pull模式都是采用消费端主动拉取的方式，即consumer轮询从broker拉取消息。
区别：
-  Push方式里，consumer把轮询过程封装了，并注册MessageListener监听器，取到消息后，唤醒MessageListener的consumeMessage()来消费，对用户而言，感觉消息是被推送过来的。
- Pull方式里，取消息的过程需要用户自己写，首先通过打算消费的Topic拿到MessageQueue的集合，遍历MessageQueue集合，然后针对每个MessageQueue批量取消息，一次取完后，记录该队列下一次要取的开始offset，直到取完了，再换另一个MessageQueue。
- 
疑问：既然是采用pull方式实现，RocketMQ如何保证消息的实时性呢？



## 长轮询

长轮询即是在请求的过程中，若是服务器端数据并没有更新，那么则将这个连接挂起，直到服务器推送新的
数据，再返回，然后进入循环周期。
客户端像传统轮询一样从服务端请求数据，服务端会阻塞请求不会立刻返回，直到有数据或超时才返回给客
户端，然后关闭连接，客户端处理完响应信息后再向服务器发送新的请求。

![image-20210106210340592](uploads/RocketMQ/RocketMQ05.png)

## 消息模式

DefaultMQPushConsumer实现了自动保存offset值以及实现多个consumer的负载均衡。

```java
//设置组名
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("HAOKE_IM");
```

通过groupname将多个consumer组合在一起，那么就会存在一个问题，消息发送到这个组后，消息怎么分配呢？
这个时候，就需要指定消息模式，分别有集群和广播模式。

### 集群模式

同一个 ConsumerGroup(GroupName相同) 里的每 个 Consumer 只消费所订阅消息的一部分内容， 同
一个 ConsumerGroup 里所有的 Consumer消费的内容合起来才是所订阅 Topic 内容的整体， 从而达到
负载均衡的目的 。

### 广播模式

同一个 ConsumerGroup里的每个 Consumer都 能消费到所订阅 Topic 的全部消息，也就是一个消息会
被多次分发，被多个 Consumer消费。

```java
// 集群模式
consumer.setMessageModel(MessageModel.CLUSTERING);
// 广播模式
consumer.setMessageModel(MessageModel.BROADCASTING);
```

### 重复消息的解决方案

造成消息重复的根本原因是：网络不可达。只要通过网络交换数据，就无法避免这个问题。所以解决这个问题的办
法就是绕过这个问题。那么问题就变成了：如果消费端收到两条一样的消息，应该怎样处理？

1. 消费端处理消息的业务逻辑保持幂等性
2. 保证每条消息都有唯一编号且保证消息处理成功与去重表的日志同时出现

- 第1条很好理解，只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样。
- 第2条原理就是利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不再处理这条消息。
- 第1条解决方案，很明显应该在消费端实现，不属于消息系统要实现的功能。
- 第2条可以消息系统实现，也可以业务端实现。
- 正常情况下出现重复消息的概率其实很小，如果由消息系统来实现的话，肯定会对消息系统的吞吐量和高可用有影响，所以
- 最好还是***由业务端自己处理消息重复的问题***，这也是RocketMQ不解决消息重复的问题的原因。
-   RocketMQ不保证消息不重复，如果你的业务需要保证严格的不重复消息，需要你自己在业务端去重。



## RocketMQ存储

RocketMQ中的消息数据存储，采用了零拷贝技术（使用 mmap + write 方式），文件系统采用 Linux Ext4 文件系
统进行存储。

### 消息数据的存储

在RocketMQ中，消息数据是保存在磁盘文件中，为了保证写入的性能，RocketMQ尽可能保证顺序写入，顺序写
入的效率比随机写入的效率高很多。
RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的，CommitLog是真正存储数据的文件，
ConsumeQueue是索引文件，存储数据指向到物理文件的配置。

![image-20210106210340592](uploads/RocketMQ/RocketMQ06.png)

如上图所示：

- 消息主体以及元数据都存储在CommitLog当中
- Consume Queue相当于kafka中的partition，是一个逻辑队列，存储了这个Queue在CommiLog中的起始
  offset，log大小和MessageTag的hashCode。
- 每次读取消息队列先读取consumerQueue,然后再通过consumerQueue去commitLog中拿到消息主体。

文件位置：

![image-20210106210340592](uploads/RocketMQ/RocketMQ07.png)

## 同步刷盘与异步刷盘

RocketMQ 为了提高性能，会尽可能地保证 磁盘的顺序写。消息在通过 Producer 写入 RocketMQ 的时候，有两
种写磁盘方式，分别是同步刷盘与异步刷盘。

- 同步刷盘
  - 在返回写成功状态时，消息已经被写入磁盘 。
  - 具体流程是：消息写入内存的 PAGECACHE 后，立刻通知刷盘线程刷盘，然后等待刷盘完成，刷盘线程
    执行完成后唤醒等待的线程，返回消息写成功的状态 。
- 异步刷盘
- 在返回写成功状态时，消息可能只是被写入了内存的 PAGECACHE，写操作的返回快，吞吐量大
  - 当内存里的消息量积累到一定程度时，统一触发写磁盘动作，快速写入。

- broker配置文件中指定刷盘方式
  - flushDiskType=ASYNC_FLUSH -- 异步
  - flushDiskType=SYNC_FLUSH -- 同步

![image-20210106210340592](uploads/RocketMQ/RocketMQ08.png)