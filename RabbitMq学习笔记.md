#  RabbitMq的简介及相关概念

 ##  什么是MQ

 MQ(message queue)，从字面意思上看，本质是个队列，FIFO 先入先出，只不过队列中存放的内容是 message 而已，还是一种跨进程的通信机制，用于上下游传递消息。在互联网架构中，MQ 是一种非常常 见的上下游“逻辑解耦+物理解耦”的消息通信服务。使用了 MQ 之后，消息发送上游只需要依赖 MQ，不 用依赖其他服务。

##  RabbitMQ 的概念 

RabbitMQ 是一个消息中间件：它接受并转发消息。你可以把它当做一个快递站点，当你要发送一个包 裹时，你把你的包裹放到快递站，快递员最终会把你的快递送到收件人那里，按照这种逻辑 RabbitMQ 是 一个快递站，一个快递员帮你传递快件。RabbitMQ 与快递站的主要区别在于，它不处理快件而是接收， 存储和转发消息数据。 

##  四大核心概念

生产者   	产生数据发送消息的程序是生产者 

交换机		 交换机是 RabbitMQ 非常重要的一个部件，一方面它接收来自生产者的消息，另一方面它将消息 推送到队列中。交换机必须确切知道如何处理它接收到的消息，是将这些消息推送到特定队列还是推 送到多个队列，亦或者是把消息丢弃，这个得有交换机类型决定 

队列  			队列是 RabbitMQ 内部使用的一种数据结构，尽管消息流经 RabbitMQ 和应用程序，但它们只能存 储在队列中。队列仅受主机的内存和磁盘限制的约束，本质上是一个大的消息缓冲区。许多生产者可 以将消息发送到一个队列，许多消费者可以尝试从一个队列接收数据。这就是我们使用队列的方式 

消费者  		消费与接收具有相似的含义。消费者大多时候是一个等待接收消息的程序。请注意生产者，消费 者和消息中间件很多时候并不在同一机器上。同一个应用程序既可以是生产者又是可以是消费者。

#  RabbitMq的工作原理

![image-20210727192509182](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727192509182.png)

工作原理:

生产者/消费者建立和RabbitMq的连接通道---------->生产者发消息到RabbitMq的==交换机==，并设置RoutingKey----------->交换机根据RoutingKey的规则选择性的把消息分发到绑定的队列中------------------>消费者从==队列==中获取数据

注意黄色字体:写代码安装这个逻辑来写

名词理解:  1RabbitMq可以简单理解成一个可以保存.分发消息的软件,程序员和可以往这个软件中发消息,取消息

​			2Connection：publisher／consumer 和 broker 之间的 TCP 连接

​			3 channel:我们往RabbitMq中发消息,实质是发二进制消息,需要建立一个通道.Channel 作为轻量级的 Connection 极大减少了操作系统建立 TCP connection 的开销

​			4 Exchange：交换机.交换机决定了RabbitMq会把消息发送到哪些队列中.

​			5Binding：exchange 和 queue 之间的虚拟连接.    

交换机怎么知道把消息发送给那些队列来接收呢？

1.通过我们发消息时设置RoutingKey。并告诉交换机跟我RoutingKey相同，或者满足一定规则的队列可以接收

2 创建队列的同时，设置RoutingKey。并绑定到交换机

3 交换机根据，生产者的RoutingKey跟队列的RoutingKey。把消息发送给指定队列

根据转发消息的不同，可以把交换机分类。常用的有3类

##  交换机类型

##  直接(direct)

![image-20210727195938237](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727195938237.png)

总结：队列中RoutingKey和发消息时设置的RoutingKey一样时，就可以接收

##  扇出(fanout)

![image-20210727200342086](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727200342086.png)

总结：只要队列和交换机绑定了就可以接收

##  主题(topic)

![image-20210727200643994](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727200643994.png)

 ![image-20210727200716844](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210727200716844.png)

##  

##  代码演示：

代码准备：  以Topic交换机为例  

工作原理:  		生产者/消费者建立和RabbitMq的连接通道---------->生产者发消息到RabbitMq的==交换机==，并设置RoutingKey----------->交换机根据RoutingKey的规则选择性的把消息分发到绑定的队列中------------------>消费者从==队列==中获取数据

```jav
<!--RabbitMQ的依赖-->
        <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>5.8.0</version>
        </dependency>

```

```java
工具类:
/**
 *连接工厂创建信道的工具类
 */
public class RabbitMqUtils {
    //得到一个连接的 channel
    public static Channel getChannel() throws Exception{
//创建一个连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("192.168.200.130");
        factory.setUsername("root");
        factory.setPassword("root");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        return channel;
    }
}
```



生产者代码

```java
public class MyProvider {
    public static final String MYExchange = "myExchange";//交换机名
    public static void main(String[] args)throws Exception {
        //自定义工具类 获取连接并创建通道
        Channel channel = RabbitMqUtils.getChannel();
        /**
         * Q1-->绑定的是
         * 中间带 orange 带 3 个单词的字符串(*.orange.*)
         * Q2-->绑定的是
         * 最后一个单词是 rabbit 的 3 个单词(*.*.rabbit)
         * 第一个单词是 lazy 的多个单词(lazy.#)
         *
         */
        Map<String, String> bindingKeyMap = new HashMap<>();
        bindingKeyMap.put("quick.orange.rabbit","被队列 Q1Q2 接收到");
        bindingKeyMap.put("lazy.orange.elephant","被队列 Q1Q2 接收到");
        bindingKeyMap.put("quick.orange.fox","被队列 Q1 接收到");
        bindingKeyMap.put("lazy.brown.fox","被队列 Q2 接收到");
        bindingKeyMap.put("lazy.pink.rabbit","虽然满足两个绑定但只被队列 Q2 接收一次");
        bindingKeyMap.put("quick.brown.fox","不匹配任何绑定不会被任何队列接收到会被丢弃");
        bindingKeyMap.put("quick.orange.male.rabbit","是四个单词不匹配任何绑定会被丢弃");
        bindingKeyMap.put("lazy.orange.male.rabbit","是四个单词但匹配 Q2");

        for (Map.Entry<String, String> bindingKeyEntry: bindingKeyMap.entrySet()){
            String bindingKey = bindingKeyEntry.getKey();
            String message = bindingKeyEntry.getValue();
            /**
             * 发送一个消息
             * 1.发送到那个交换机
             * 2.RoutingKey
             * 3.其他的参数信息 比如这个消息的存活时间,是否持久化到队列等
             * 4.发送消息的消息体
             */
            channel.basicPublish(MYExchange,bindingKey, null, message.getBytes("UTF-8"));
            System.out.println("生产者发出消息" + message);
        }


        System.out.println("消息发送完毕");
    }
}
```

消息接收者1

```java
public class MyReceive {
    public static final String MYExchange = "myExchange";//交换机
    public static final String MYQUEUE = "Q1";//队列

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        /**
         * 声明交换机
         *1 交换机名
         * 2交换机类型
         * 3交换机是否持久化
         * 3 交换机是否自动删除
         * 4 其他信息
         */
        channel.exchangeDeclare(MYExchange, BuiltinExchangeType.TOPIC, true, true, null);
        /**
         * 生成一个队列
         * 1.队列名称
         * 2.队列里面的消息是否持久化 默认消息存储在内存中
         * 3.该队列是否只供一个消费者进行消费 是否进行共享 true 可以多个消费者消费
         * 4.是否自动删除 最后一个消费者端开连接以后 该队列是否自动删除 true 自动删除
         * 5.其他参数  队列的一些属性信息,例如队列中数据过期时间,队列中数据被拒收之后怎么处理等 数据类型是Map<String, Object> arguments
         */
        channel.queueDeclare(MYQUEUE, true, false, true, null);
        //队列绑定交换机
        channel.queueBind(MYQUEUE, MYExchange, "*.orange.*", null);
        //推送的消息如何进行消费的接口回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody());
            System.out.println(message);
        };
        //取消消费的一个回调接口 如在消费的时候队列被删除掉了
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消息消费被中断");
        };
        /**
         * 消费者消费消息
         * 1.消费哪个队列
         * 2.消费成功之后是否要自动应答 true 代表自动应答 false 手动应答
         * 3.消费者接收数据后的回调
         * 4 取消消费的回调
         */
        channel.basicConsume(MYQUEUE, true, deliverCallback, cancelCallback);
    }
}
```

消息接收者2

```java
public class MyReceive02 {
    public static final String MYExchange = "myExchange";
    public static final String MYQUEUE = "Q2";

    public static void main(String[] args) throws Exception {
        Channel channel = RabbitMqUtils.getChannel();
        //声明交换机
        channel.exchangeDeclare(MYExchange, BuiltinExchangeType.TOPIC, true, true, null);
        //声明队列
        channel.queueDeclare(MYQUEUE, true, false, true, null);
        //队列和交换机绑定  绑定了两个RoutingKey
        channel.queueBind(MYQUEUE, MYExchange, "*.*.rabbit", null);
        channel.queueBind(MYQUEUE, MYExchange, "lazy.#", null);
        //推送的消息如何进行消费的接口回调
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody());
            System.out.println(message);
        };
        //取消消费的一个回调接口 如在消费的时候队列被删除掉了
        CancelCallback cancelCallback = (consumerTag) -> {
            System.out.println("消息消费被中断");
        };
        //接受获取消息
        channel.basicConsume(MYQUEUE, true, deliverCallback, cancelCallback);
    }

}
```

运行结果如下

```java
MyReceive:
被队列 Q1Q2 接收到
被队列 Q1Q2 接收到
被队列 Q1 接收到
被队列 Q1Q2 接收到
被队列 Q1Q2 接收到
被队列 Q1 接收到
MyReceive02
被队列 Q1Q2 接收到
被队列 Q2 接收到
被队列 Q1Q2 接收到
虽然满足两个绑定但只被队列 Q2 接收一次
是四个单词但匹配 Q2
```

#  SpringBoot集成RabbitMq

##  RabbitMq持久化和发布确认机制

##  RabbitMq集群搭建

![image-20210728034418118](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210728034418118.png)

##  暂时不懂的问题,回头补



