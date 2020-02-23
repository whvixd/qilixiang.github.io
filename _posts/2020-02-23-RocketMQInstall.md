---
layout:     post
title:      RocketMQ Install
subtitle:   mq
date:       2020-02-23
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - mq
---

> RocketMQ相关知识
参考 [https://blog.csdn.net/javahongxi/article/details/84931747](https://blog.csdn.net/javahongxi/article/details/84931747)

#### CentOS6 上安装RocketMQ


> CentOS上安装 java git mvn 可参考 [https://www.cnblogs.com/fly-piglet/p/7594488.html](https://www.cnblogs.com/fly-piglet/p/7594488.html)

##### 1. 克隆RocketMQ项目代码

```
git clone https://github.com/apache/rocketmq.git #克隆项目
cd rocketmq #进入项目
git branch -r #查看线上分支
git checkout 'release-4.6.1' # 切换到release-4.6.1，可自行切换分支
mvn -Prelease-all -DskipTests clean install -U #编译项目，下载依赖jar包，生成可执行命令
cd distribution/target/rocketmq-4.6.1/rocketmq-4.6.1/bin/ #进行bin文件
```

##### 2. 启动namesrv、broker
```
nohup sh mqnamesrv & #启动mqnamesrv
```
> 可能会启动失败，打开nohub.log日志，提示内存不够，Native memory allocation (mmap) failed to map 2147483648 bytes for committing reserved memory.

```
vi runserver.sh #修改JVN堆栈大小，-server -Xms4g -Xmx4g -Xmn2g 改为 -server -Xms400m -Xmx400m -Xmn200m
vi runbroker.sh #类似，可根据自己机器大小配置
```

```
nohup sh mqnamesrv & #启动mqnamesrv
nohup sh mqbroker -n localhost:9876 autoCreateTopicEnable=true & #启动broker，autoCreateTopicEnable=true 可自定义topic
ps -ef | grep java #检查 namesrv、broker进程是否启动成功
```

##### 3.关闭namesrv、broker
```
sh mqshutdown broker #关闭broker
sh mqshutdown namesrv #关闭namesrv
```

##### 4. 配置开机自启动namesrv、broker

```
vi mq_start.sh #创建mq启动文件
vi mq_broker_start.sh #创建broker启动文件
```
> 目录根据自己机器修改

**mq_broker_start.sh：**
```
#!/bin/bash
sh /home/xxx/software/middleware/mq/rocketmq/distribution/target/rocketmq-4.6.1/rocketmq-4.6.1/bin/mqbroker -n localhost:9876 autoCreateTopicEnable=true
```

**mq_start.sh：**

```
#!/bin/bash
export JAVA_HOME=/home/xxx/software/Language/Java/jdk1.8.0_181 #配置JAVA_HOME环境变量
nohup sh /home/xxx/software/middleware/mq/rocketmq/distribution/target/rocketmq-4.6.1/rocketmq-4.6.1/bin/mqnamesrv 1>/home/xxx/logs/rocketmqlogs/mqnamesrv_`date +%Y``date +%m``date +%d`.log & # 启动namesrv，并日志输出

nohup sh /home/xxx/shell/mq_broker_start.sh 1>/home/xxx/logs/rocketmqlogs/mqbroker_`date +%Y``date +%m``date +%d`.log & #启动broker
```

```
sudo -i #切换到root
vi /etc/rc.d/rc.local #编辑 启动文件

```

**rc.local：**
```
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
sh /home/xxx/shell/mq_start.sh # 启动mq命令
```

##### 5. 测试
```
// 发送
public class Producer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer defaultMQProducer = new DefaultMQProducer("defaultMQPushConsumerGroup");
        defaultMQProducer.setNamesrvAddr("xxx.xxx.xxx.xxx:9876"); // ip地址
        defaultMQProducer.start();
        for (int i = 0; i < 100; i++) {
            Message message = new Message("TopicA", "A", ("Hello Mq" + i).getBytes("UTF-8"));
            SendResult sendResult = defaultMQProducer.send(message);
            System.out.printf("%s%n", sendResult);
        }
        defaultMQProducer.shutdown();
    }
}

//消费
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer defaultMQPushConsumer = new DefaultMQPushConsumer("defaultMQPushConsumerGroup");
        defaultMQPushConsumer.setNamesrvAddr("xxx.xxx.xxx.xxx:9876");
        defaultMQPushConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        defaultMQPushConsumer.subscribe("TopicA", "*");//subExpression订阅某个tag。* 为所有的tag
        defaultMQPushConsumer.registerMessageListener((MessageListenerConcurrently) (messages, context) -> {
            messages.forEach(message -> System.out.println(new String(message.getBody())));
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });

        defaultMQPushConsumer.start();
    }
}
```