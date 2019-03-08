#### 1、特点

```
RocketMQ是一款分布式、队列模型的消息中间件，具有以下特点：

    支持严格的消息顺序
    支持Topic与Queue两种模式
    亿级消息堆积能力
    比较友好的分布式特性
    同时支持Push与Pull方式消费消息
    历经多次天猫双十一海量消息考验
```

```
rocketmq中的所有消息都是持久化的，先写入系统pagecache，
然后刷盘，可以保证内存与磁盘都有一份数据，访问时，可以直接从内存读取
```

```
目前主流的MQ主要是Rocketmq、kafka、Rabbitmq，Rocketmq相比于Rabbitmq、kafka具有主要优势特性有：
•       支持事务型消息（消息发送和DB操作保持两方的最终一致性，rabbitmq和kafka不支持）
•       支持结合rocketmq的多个系统之间数据最终一致性（多方事务，二方事务是前提）
•       支持18个级别的延迟消息（rabbitmq和kafka不支持）
•       支持指定次数和时间间隔的失败消息重发（kafka不支持，rabbitmq需要手动确认）
•       支持consumer端tag过滤，减少不必要的网络传输（rabbitmq和kafka不支持）
•       支持重复消费（rabbitmq不支持，kafka支持）
```
#### rockmq发送状态

```
package com.alibaba.rocketmq.client.producer;

public enum SendStatus {
    SEND_OK, //消息发送成功
    FLUSH_DISK_TIMEOUT,//消息发送成功，但是服务器刷盘超时，消息已经进入服务器队列，只有此时服务器宕机，消息才会丢失
    FLUSH_SLAVE_TIMEOUT,//消息发送成功，但是服务器同步到Slave时超时，消息已经进入服务器队列，只有此时服务器宕机，消息才会丢失
    SLAVE_NOT_AVAILABLE;//消息发送成功，但是此时slave不可用，消息已经进入服务器队列，只有此时服务器宕机，消息才会丢失

    private SendStatus() {
    }
}
```
#### RocketMq有3中消息类型
- 普通消费
- 顺序消费
- - 刚刚下单：第一、创建订单 ，第二：订单付款，第三：订单完成，环节要有顺序，这个订单才有意义
- - -
如何保证顺序？
> produce在发送消息的时候，把消息发到同一个队列（queue）中,消费者注册消息监听器为MessageListenerOrderly（实现此表示一个队列只会被一个线程取到，第二个线程无法访问这个队列 ），这样就可以保证消费端只有一个线程去消费消息
>>注意：是把把消息发到同一个队列（queue），不是同一个topic，默认情况下一个topic包括4个queue
- - - 
- 事务消费

```
public class Producer {  
    public static void main(String[] args) throws MQClientException, InterruptedException {  
        // 未决事务，MQ服务器回查客户端
        // 也就是上文所说的，当RocketMQ发现`Prepared消息`时，会根据这个Listener实现的策略来决断事务
        TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();  
        TransactionMQProducer producer = new TransactionMQProducer("transaction_Producer");  
        producer.setNamesrvAddr("192.168.100.145:9876;192.168.100.146:9876;192.168.100.149:9876;192.168.100.239:9876");  
        // 事务回查最小并发数  
        producer.setCheckThreadPoolMinSize(2);  
        // 事务回查最大并发数  
        producer.setCheckThreadPoolMaxSize(2);  
        // 队列数  
        producer.setCheckRequestHoldMax(2000);  
        producer.setTransactionCheckListener(transactionCheckListener);  
        producer.start();  
  
       // 本地事务的处理逻辑，相当于示例中检查Bob账户并扣钱的逻辑
        TransactionExecuterImpl tranExecuter = new TransactionExecuterImpl();  
        for (int i = 1; i <= 2; i++) {  
            try {  
                Message msg = new Message("TopicTransactionTest", "transaction" + i, "KEY" + i,  
                        ("Hello RocketMQ " + i).getBytes());  
                SendResult sendResult = producer.sendMessageInTransaction(msg, tranExecuter, null);  
                System.out.println(sendResult);  
  
                Thread.sleep(10);  
            } catch (MQClientException e) {  
                e.printStackTrace();  
            }  
        }  
  
        for (int i = 0; i < 100000; i++) {  
            Thread.sleep(1000);  
        }  
  
        producer.shutdown();  
  
    }  
}  
```

```
/** 
 * 执行本地事务 
 */  
public class TransactionExecuterImpl implements LocalTransactionExecuter {  
    // private AtomicInteger transactionIndex = new AtomicInteger(1);  
  
    public LocalTransactionState executeLocalTransactionBranch(final Message msg, final Object arg) {  
  
        System.out.println("执行本地事务msg = " + new String(msg.getBody()));  
  
        System.out.println("执行本地事务arg = " + arg);  
  
        String tags = msg.getTags();  
  
        if (tags.equals("transaction2")) {  
            System.out.println("======我的操作============，失败了  -进行ROLLBACK");  
            return LocalTransactionState.ROLLBACK_MESSAGE;  
        }  
        return LocalTransactionState.COMMIT_MESSAGE;  
        // return LocalTransactionState.UNKNOW;  
    }  
}
```

#### rockmq消费策略

```
 //这里设置的是一个consumer的消费策略
//CONSUME_FROM_LAST_OFFSET 默认策略，从该队列最尾开始消费，即跳过历史消息
//CONSUME_FROM_FIRST_OFFSET 从队列最开始开始消费，即历史消息（还储存在broker的）全部消费一遍
//CONSUME_FROM_TIMESTAMP 从某个时间点开始消费，和setConsumeTimestamp()配合使用，默认是半个小时以前
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
```
#### rocketMQ默认是集群消费,我们可以通过在Consumer来支持广播消费

```
consumer.setMessageModel(MessageModel.BROADCASTING);// 广播消费
```

