---
title: RocketMQ服务的搭建和使用
tags: RocketMQ
categories: 中间件
---

##   简述
   在分布式应用开发过程中，有一个非常重要的中间件RocketMQ。在项目的解耦，与削峰，或者一些非实时的订单生成，用户一些信息更新
    等等地方都能够看到它的身影，这篇就记录下怎么搭建使用它...

<!-- more -->

##  为什么要使用

rocketmq是阿里巴巴在2012年开源的分布式消息中间件，并且已经捐赠给Apache软件基金会。所以为啥用呢？它抗过双十一啊，
下面会记录下它的一些特点：
*   可扩展
    RocketMQ的四大组件，Name Server、Broker、Producter、Cpnsumer等都可以平扩展

*   海量消息堆积能力
    采用零拷贝原来实现超大消息堆积能力

*   支持顺序消息

*   多种消息过滤方式
    消费端过滤/生产端过滤

*   支持事务

*   回溯消费
    已经消费成功的消息在业务需求上可重新消费

##  搭建
###   windows环境
*    [下载]("http://rocketmq.apache.org/dowloading/releases/")，下载二进制Binary

*   由于本地内存屡屡受挫，这里将bin下的runserver.cmd和runbroker.cmd中的Xms，Xmx，Xmn更改为256m,256m,128m。

*   新建系统变量ROCKETMQ_HOME C:\XXX\rocketmq

*   启动，cmd进到bin目录下
    *   start mqnamesrv.cmd
    启动成功如下：
    ```
    Java HotSpot(TM) 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
    Java HotSpot(TM) 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
    The Name Server boot success. serializeType=JSON
    ```

    *   start mqbroker.cmd -n localhost:9876
        *   此处若出现找不到或无法加载主类，可查看修改runbroker.cmd，将%CLASSPATH%加上双引号
        *   启动成功如下：
        ```
        The broker[LAPTOP-PKGRVCGD, 10.8.0.8:10911] boot success. serializeType=JSON and name server is localhost:9876
        ```

    *   start mqadmin.cmd updateTopic -n localhost:9876 -b 10.8.0.8:10911 -t demo
    这里的10.8.0.8是在启动broker时显示的。

    以上都成功后就可使用下一节的demo进行调试了。


##  代码demo
*   首先引入jar包
```
    compile 'org.apache.rocketmq:rocketmq-client:4.2.0'
```

*   编写启动produc
```
    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer defaultMQProducer = new DefaultMQProducer("defaultGroup");

        try {
            //namespace
            defaultMQProducer.setNamesrvAddr("localhost:9876");
            defaultMQProducer.setRetryTimesWhenSendFailed(3);
            defaultMQProducer.start();

            for (int i = 0; i < 5; i++) {
                Message msg = new Message(
                        "demo",
                        "tagA",
                        ("hello, duanshaojie" + i ).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = defaultMQProducer.send(msg);
                System.out.println("result------------>" + JSON.toJSONString(sendResult));
            }
        }finally {
            defaultMQProducer.shutdown();
        }
    }
```
成功启动打印如下:
```
result------------>{"messageQueue":{"brokerName":"LAPTOP-PKGRVCGD","queueId":2,"topic":"demo"},"msgId":"C0A807374A5018B4AAC2078FB9190000","offsetMsgId":"0A08000800002A9F0000000000000B97","queueOffset":0,"regionId":"DefaultRegion","sendStatus":"SEND_OK","traceOn":true}
result------------>{"messageQueue":{"brokerName":"LAPTOP-PKGRVCGD","queueId":3,"topic":"demo"},"msgId":"C0A807374A5018B4AAC2078FB92C0001","offsetMsgId":"0A08000800002A9F0000000000000C5E","queueOffset":1,"regionId":"DefaultRegion","sendStatus":"SEND_OK","traceOn":true}
result------------>{"messageQueue":{"brokerName":"LAPTOP-PKGRVCGD","queueId":4,"topic":"demo"},"msgId":"C0A807374A5018B4AAC2078FB92E0002","offsetMsgId":"0A08000800002A9F0000000000000D25","queueOffset":0,"regionId":"DefaultRegion","sendStatus":"SEND_OK","traceOn":true}
result------------>{"messageQueue":{"brokerName":"LAPTOP-PKGRVCGD","queueId":5,"topic":"demo"},"msgId":"C0A807374A5018B4AAC2078FB9310003","offsetMsgId":"0A08000800002A9F0000000000000DEC","queueOffset":0,"regionId":"DefaultRegion","sendStatus":"SEND_OK","traceOn":true}
result------------>{"messageQueue":{"brokerName":"LAPTOP-PKGRVCGD","queueId":6,"topic":"demo"},"msgId":"C0A807374A5018B4AAC2078FB9330004","offsetMsgId":"0A08000800002A9F0000000000000EB3","queueOffset":0,"regionId":"DefaultRegion","sendStatus":"SEND_OK","traceOn":true}
```

*   编写consumer
```
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("defaultGroup");

        consumer.setNamesrvAddr("localhost:9876");
        consumer.subscribe("demo", "tagA");

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.println(new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
    }
```

获取消息日志如下:
```
hello, duanshaojie1
hello, duanshaojie3
hello, duanshaojie0
hello, duanshaojie4
hello, duanshaojie2
```