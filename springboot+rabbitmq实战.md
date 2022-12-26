---
title: springboot+rabbitmq实战
date: 2019-08-11
updated: 2019-08-11
tags: [springboot, rabbitmq, queue]
---

安装
====

archlinux 安装启动
------------------

-   yaourt -S rabbitmq
-   systemctl start rabbitmq.service
-   测试服务启动
    -   rabbitmqctl status
-   安装管理客户端
    -   rabbitmq-plugins enable rabbitmq~management~
    -   web访问地址 <http://127.0.0.1:15672>

centos(阿里ecs)
---------------

-   yaourt -S rabbitmq-server
-   systemctl start rabbitmq-server.service
-   测试服务启动
-   安装管理客户端
    -   rabbitmq-plugins enable rabbitmq~management~
    -   web访问地址 <http://47.75.36.192:15672>
-   参考: <https://www.rabbitmq.com/install-rpm.html>

配置
====

增加用户并授权
--------------

1.  rabbitmqctl add~user~ admin 123456 \# 增加用户并设置用户名密码
2.  rabbitmqctl set~usertags~ admin administrator \# 授权
3.  rabbitmqctl set~permissions~ -p / admin \'.\*\' \'.\*\' \'.\*\' \#
    不知道干啥的但是需要不然访问不了

常用操作
========

相关概念
========

通常我们谈到队列服务, 会有三个概念： 发消息者、队列、收消息者，RabbitMQ
在这个基本概念之上, 多做了一层抽象, 在发消息者和 队列之间, 加入了交换器
(Exchange). 这样发消息者和队列就没有直接联系,
转而变成发消息者把消息给交换器, 交换器根据调度策略再把消息再给队列。

虚拟主机
--------

一个虚拟主机持有一组交换机、队列和绑定。为什么需要多个虚拟主机呢？很简单，
RabbitMQ 当中，用户只能在虚拟主机的粒度进行权限控制。
因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。每一个
RabbitMQ 服务器都有一个默认的虚拟主机"/"。

交换机(Exchange)
----------------

交换机的功能主要是接收消息并且转发到绑定的队列，交换机不存储消息，在启用ack模式后，交换机找不到队列会返回错误。交换机有四种类型：Direct,
topic, Headers and Fanout Direct：direct 类型的行为是"先匹配, 再投送".
即在绑定时设定一个 routing~key~, 消息的routing~key~ 匹配时,
才会被交换器投送到绑定的队列中去，　默认方式
Topic：按规则转发消息（最灵活）, 主要通过通配符 Headers：设置 header
attribute 参数类型的交换机 Fanout：转发消息到所有绑定队列

springboot整合
==============

redirexchange 默认
------------------

1.  引入依赖

    ``` {.xml}
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    ```

2.  配置rabbit地址, application.yaml

    ``` {.yaml}
    spring:
      rabbitmq:
        host: localhost
        password: guest
        port: 5672
        username: guest

    ```

3.  增加queue配置

    ``` {.java}
    @Configuration
    public class RabbitConfig {

        /**
        * 创建一个叫hello的queue
        * @return
        */
        @Bean
        public Queue queue(){
            return new Queue("ttang");
        }
    }

    ```

4.  producer

    ``` {.java}
    @Component
    @Slf4j
    public class RabbitProducer {

        @Autowired
        private RabbitTemplate rabbitTemplate;

        public void sendMsgString(){
            String context = "ttang say: " + new Date();
            log.info(context);
            rabbitTemplate.convertAndSend("ttang", context);
        }

        public void sendMsgInt(int i){
            log.info("{}, {}", Thread.currentThread().getStackTrace()[1].getMethodName(), i);
            rabbitTemplate.convertAndSend("ttang", i);
        }
    }

    ```

5.  consumer

    ``` {.java}
    /**
    *
    * 接收消息
    *
    */
    @Component
    @RabbitListener(queues = "ttang")
    @Slf4j
    public class RabbitConsumer {

        @RabbitHandler
        public void receiveMsgString(String tt){
            log.info("收到ttang消息, {}", tt);
        }

        /**
        * Caused by: org.springframework.amqp.AmqpException: Ambiguous methods for payload type: class java.lang.String:
        * msg and msg2
        * 1. 不能有两个payload同时匹配, 会根据消息类型自动匹配
        * @param tt
        */
        @RabbitHandler
        public void receiveMsgInt(Integer tt){
            log.info("收到ttang消息2, {}", tt);
        }
    }

    ```

6.  test

    ``` {.java}
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class RabbitConsumerTest {

        @Autowired
        private RabbitProducer producer;

        @Test
        public void msg() {
    //        final ExecutorService fixedThreadPool = Executors.newFixedThreadPool(5);
    //        for (int i = 0; i < 10; i++) {
    //            fixedThreadPool.execute(() -> {
    //                producer.sendMsgString();
    //            });
    //        }

            for (int i = 0; i < 10; i++) {
                producer.sendMsgInt(i);
                producer.sendMsgString();
            }
        }
    }
    ```

topicexchange 模式匹配订阅
--------------------------

依赖、地址同上

1.  queue,exchange配置,　配置路由规则

    ``` {.java}
    /**
    * 需要三个bean实例 queue, exchange, bindexchange
    *
    * queue1比queue2只有一个字母之差，模拟匹配
    */
    @Configuration
    public class TopicExchangeConfig {

        /**
        * 定制主题
        * @return
        */
        @Bean
        public TopicExchange topicExchange(){
            return new TopicExchange("topicExchange");
        }

        @Bean
        public Queue queueMessage(){
            return new Queue("topic.message");
        }

        /**
        * 精确匹配topic.message
        * @param queueMessage
        * @param topicExchange
        * @return
        */
        @Bean
        Binding bindingMessage(Queue queueMessage, TopicExchange topicExchange){
            return BindingBuilder.bind(queueMessage).to(topicExchange).with("topic.message");
        }

        @Bean
        public Queue queueMessages(){
            return new Queue("topic.messages");
        }

        /**
        * 匹配多个
        * *表示一个词.
        * #表示零个或多个词.
        *
        * 这个queue能接收多个消息
        * @param queueMessages
        * @param topicExchange
        * @return
        */
        @Bean
        Binding bindingMessages(Queue queueMessages, TopicExchange topicExchange){
            return BindingBuilder.bind(queueMessages).to(topicExchange).with("topic.#");
        }
    }

    ```

2.  生产者

    ``` {.java}
    @Component
    @Slf4j
    public class TopicExchangeProducer {

        @Autowired
        private RabbitTemplate rabbitTemplate;

        /**
        * 同时匹配 topic.message和 topic.# 所以会发到这两个queue里
        * @param msg
        */
        public void sendMessage(String msg){
            final StackTraceElement element = Thread.currentThread().getStackTrace()[1];
            log.info("{}, {}", element.getMethodName(), msg);
            rabbitTemplate.convertAndSend("topicExchange", "topic.message"
                    , String.join(",", Arrays.asList(element.getClassName(), element.getMethodName(), msg)));
        }

        /**
        * 只能匹配到topic.#, 多了个s, 所以只会匹配发送到 queuemessages
        * BindingBuilder.bind(queueMessages).to(topicExchange).with("topic.#");
        * 相应的接收也只会是topic.messages
        * @param msg
        */
        public void sendMessages(String msg){
            final StackTraceElement element = Thread.currentThread().getStackTrace()[1];
            log.info("{}, {}", element.getMethodName(), msg);
            rabbitTemplate.convertAndSend("topicExchange", "topic.messages"
                    , String.join(",", Arrays.asList(element.getClassName(), element.getMethodName(), msg)));
        }
    }

    ```

3.  消费者

    ``` {.java}
    /**
    * 监听topic.message
    */
    @Slf4j
    @Component
    @RabbitListener(queues = "topic.message")
    public class TopicConsumerMessage {

      @RabbitHandler
      public void receiveMsgString(String msg){
          final StackTraceElement element = Thread.currentThread().getStackTrace()[1];
          log.info("{}, {}", element.getMethodName(), msg);
      }
    }

    /**
    * 监听topic.message
    */
    @Slf4j
    @Component
    @RabbitListener(queues = "topic.messages")
    public class TopicConsumerMessages {

      @RabbitHandler
      public void receiveMsgString(String msg){
          final StackTraceElement element = Thread.currentThread().getStackTrace()[1];
          log.info("{}, {}", element.getMethodName(), msg);
      }
    }

    ```

4.  测试

    ``` {.java}
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class TopicExchangeProducerTest {

      @Autowired
      private TopicExchangeProducer topicExchangeProducer;

      /**
      * 2019-08-11 12:54:46.482  INFO 18374 --- [           main] c.l.d.s.t.p.TopicExchangeProducer        :
      * sendMessage, 我爱你　糖糖
      * <p>
      * <p>
      * 2019-08-11 12:54:46.502  INFO 18374 --- [ntContainer#2-1] c.l.d.s.t.c.TopicConsumerMessages        :
      * receiveMsgString, com.lx.demo.springbootrabbitmq.tpcexchange.producer.TopicExchangeProducer,sendMessage,我爱你　糖糖
      * 2019-08-11 12:54:46.503  INFO 18374 --- [ntContainer#1-1] c.l.d.s.t.consumer.TopicConsumerMessage  :
      * receiveMsgString, com.lx.demo.springbootrabbitmq.tpcexchange.producer.TopicExchangeProducer,sendMessage,我爱你　糖糖
      * 这里producer发送用的routing_key topic.message匹配topic.message和topic.#, 所以可以干到两个queue里, 相应的消费两个queue的都能接收到
      */
      @Test
      public void sendMessage() {
          topicExchangeProducer.sendMessage("我爱你　糖糖");
      }

      /**
      * 2019-08-11 12:57:00.110  INFO 18687 --- [           main] c.l.d.s.t.p.TopicExchangeProducer        :
      * sendMessages, i love you, ttang
      * <p>
      * <p>
      * 2019-08-11 12:57:00.128  INFO 18687 --- [       Thread-2] o.s.a.r.l.SimpleMessageListenerContainer : Waiting
      * for workers to finish.
      * 2019-08-11 12:57:00.128  INFO 18687 --- [ntContainer#2-1] c.l.d.s.t.c.TopicConsumerMessages        :
      * receiveMsgString, com.lx.demo.springbootrabbitmq.tpcexchange.producer.TopicExchangeProducer,sendMessages,i
      * love you, ttang
      * messages的消息发送者用的routing_key只能匹配topic.#, 所以只能到topic.messages这个queue
      */
      @Test
      public void sendMessages() {
          topicExchangeProducer.sendMessages("i love you, ttang");
      }
    } 
    ```

fanoutexchange 发布订阅
-----------------------

依赖、地址同上

1.  queue配置

    ``` {.java}
    /**
    * 定义多个queue, 一个Fanoutexchange, 配置绑定binding
    * 这样发送到交换机上的消息都会传递到所有绑定queue
    *
    * 这个最像传统的主题订阅模式
    */
    @Configuration
    public class FanoutExchangeConfig {

    /**
      * 创建交换机
      * @return
      */
    @Bean
    FanoutExchange fanoutExchange(){
        return new FanoutExchange("fanoutExchange");
    }

    /**
      * 创建queue
      * @return
      */
    @Bean
    Queue fanoutQueue1(){
        return new Queue("fanout.1");
    }

    /**
      * 这里不在需要指定binding_key, 就算配了也么得用
      * @param fanoutQueue1
      * @param fanoutExchange
      * @return
      */
    @Bean
    Binding binding1(Queue fanoutQueue1, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }

    @Bean
    Queue fanoutQueue2(){
        return new Queue("fanout.2");
    }

    @Bean
    Binding binding2(Queue fanoutQueue2, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }

    @Bean
    Queue fanoutQueue3(){
        return new Queue("fanout.3");
    }

    /**
      * Description:
      *
      * Parameter 0 of method binding1 in com.lx.demo.springbootrabbitmq.fanoutexchange.config.FanoutExchangeConfig
      * required a single bean, but 6 were found:
      * 	- queue: defined by method 'queue' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/direxchange/config/RabbitConfig.class]
      * 	- fanoutQueue1: defined by method 'fanoutQueue1' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/fanoutexchange/config/FanoutExchangeConfig.class]
      * 	- fanoutQueue2: defined by method 'fanoutQueue2' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/fanoutexchange/config/FanoutExchangeConfig.class]
      * 	- fanoutQueue3: defined by method 'fanoutQueue3' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/fanoutexchange/config/FanoutExchangeConfig.class]
      * 	- queueMessage: defined by method 'queueMessage' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/tpcexchange/config/TopicExchangeConfig.class]
      * 	- queueMessages: defined by method 'queueMessages' in class path resource
      * 	[com/lx/demo/springbootrabbitmq/tpcexchange/config/TopicExchangeConfig.class]
      *
      * @param fanoutQueue3
      * @param fanoutExchange
      * @return
      */
    @Bean
    Binding binding3(Queue fanoutQueue3, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue3).to(fanoutExchange);
    }

    }

    ```

2.  生产者

    ``` {.java}
    @Component
    @Slf4j
    public class FanoutExchangeProducer {

        @Autowired
        private RabbitTemplate rabbitTemplate;

        /**
        * fanout的方式不再需要routing-key
        * @param msg
        */
        public void sendMsg(String msg){
            log.info("{}, {}", Thread.currentThread().getStackTrace()[1].getMethodName(), msg);
            // 这里一定要注意函数重载，只有三个参数时，第一个才是exchange
            rabbitTemplate.convertAndSend("fanoutExchange","", msg);
        }
    }

    ```

    **注意，　发送时是三个参数**

3.  消费者

    ``` {.java}
    @Slf4j
    @Component
    @RabbitListener(queues = "fanout.1")
    public class FanoutConsumer1 {

        @RabbitHandler
        public void receiveMessageString(String msg){
            log.info("{}, {}", Thread.currentThread().getStackTrace()[1].getMethodName(), msg);
        }
    }

    ```

4.  测试

    ``` {.java}
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class FanoutExchangeProducerTest {

        @Autowired
        private FanoutExchangeProducer fanoutExchangeProducer;

        /**
        * 2019-08-11 13:48:16.402  INFO 26183 --- [           main] c.l.d.s.f.p.FanoutExchangeProducer       : sendMsg,
        * 大渣吼，我是范二特
        *
        *
        * 2019-08-11 13:48:16.425  INFO 26183 --- [ntContainer#2-1] c.l.d.s.f.consumer.FanoutConsumer2       :
        * receiveMessageString, 大渣吼，我是范二特
        * 2019-08-11 13:48:16.425  INFO 26183 --- [ntContainer#1-1] c.l.d.s.f.consumer.FanoutConsumer1       :
        * receiveMessageString, 大渣吼，我是范二特
        * 2019-08-11 13:48:16.425  INFO 26183 --- [ntContainer#3-1] c.l.d.s.f.consumer.FanoutConsumer3       :
        * receiveMessageString, 大渣吼，我是范二特
        *
        * 发送一条消息，上述三个消费者都收到了消息
        */
        @Test
        public void sendMsg() {
            fanoutExchangeProducer.sendMsg("大渣吼，我是范二特");
        }
    } 
    ```

完整代码
--------

<https://github.com/microzhao/demo/tree/master/springboot/springboot-rabbitmq>
