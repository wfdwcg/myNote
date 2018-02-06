*https://www.aliyun.com/ss/?k=RabbitMQ%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97*

*使用配置：
user.rabbitmq.consumer.addresses=a.b.c.d:xxxx
user.rabbitmq.consumer.virtualHost=/aaaa
user.rabbitmq.consumer.username=bbbb
user.rabbitmq.consumer.password=cccc
user.rabbitmq.consumer.exchange=dddd
user.rabbitmq.consumer.queue=eee_queue
*

* queue自行制定，在系统启动时会自动关联到exchange
* exchange手动在MQ控制台创建，并手动设置与queue的绑定
* 一个queue里的多个机器共享同一份消息队列的数据（一个队列的消息可能发到多个机器中的一个）

*RabbitMQ 的Messaging Model就是Producer并不会直接发送Message到queue。实际上，Producer并不知道它发送的Message是否已经到达queue。Producer发送的Message实际上是发到了Exchange中。它的功能也很简单：从Producer接收Message，然后投递到queue中。Exchange需要知道如何处理Message，是把它放到那个queue中，还是放到多个queue中？这个rule是通过Exchange 的类型定义的。*
*从AMQP协议可以看出，MessageQueue、Exchange和Binding构成了AMQP协议的核心，生产者在发送消息时，都需要指定一个RoutingKey和Exchange，Exchange在接到该RoutingKey以后，会判断该ExchangeType*

**Fanout Exchange （广播模式）
  不处理RoutingKey。你只需要简单的将queue绑定到Exchange。一个发送到Exchange的消息都会被转发到与该Exchange绑定的所有queue上。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。 **
*Direct Exchange 
  需要将一个queue绑定到Exchange上，要求该消息与RoutingKey完全匹配。这是完整的匹配。如果一个队列绑定到该Exchange上要求RoutingKey “test”，则只有被标记为“test”的消息才被转发，不会转发test.aaa，也不会转发dog.123。*
*Topic Exchange 
  将RoutingKey和某模式进行匹配。此时queue需要绑定要一个模式上。符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词。因此“audit.#”能够匹配到“audit.irs.corporate”，但是“audit.*” 只会匹配到“audit.irs”。*

  *绑定其实就是关联了exchange和queue。或者这么说：queue对exchagne的内容感兴趣，exchange要把它的Message deliver到queue中。对于fanout的exchange来说，这个参数是被忽略的。Direct exchange的路由算法非常简单：通过binding key的完全匹配。*

*如果有多个消费者同时订阅同一个队列的话，RabbitMQ是采用循环的方式分发消息的，每一条消息只能被一个订阅者接收。例如，有队列Queue，其中ClientA和ClientB都Consume了该队列，MessageA到达队列后，被分派到ClientA，ClientA回复服务器收到响应，服务器删除MessageA；再有一条消息MessageB抵达队列，服务器根据“循环推送”原则，将消息会发给ClientB，然后收到ClientB的确认后，删除MessageB；等到再下一条消息时，服务器会再将消息发送给ClientA。*

*支持事务是AMQP协议的重要特性。假设当生产者将一个持久化消息发给服务器时，因为consume命令本身没有Response返回，即使服务器崩溃生产者也无法获知该消息已丢失。如果使用事务，通过txSelect()开启一个事务，然后发送消息给服务器，然后通过txCommit()提交该事务，即可保证如果txCommit()提交了，则该消息一定会持久化，如果txCommit()未提交，则该消息不会服务器接收。当然Rabbit MQ也提供txRollback()用于回滚事务。*

*如果用标准AMQP协议，则唯一保证消息不丢失是利用事务机制 -- 令 channel 处于 transactional 模式、向其 publish 消息、执行 commit 动作。这时事务机制会带来大量开销，导致吞吐量下降 250% 。为补救事务带来的问题，引入confirmation 机制（即 Publisher Confirm）。*

*Confirm机制：使用事务可以保证只有提交的事务才会被服务器执行。但同时也将客户端与消息服务器同步起来，背离了消息队列解耦的本质。Rabbit MQ提供更轻量的机制保证生产者感知消息是否被路由到正确队列——Confirm。如果设置channel为confirm状态，则通过该channel发送的消息都被分配一个唯一ID，一旦该消息被正确路由到匹配队列，服务器返回生产者一个Confirm，包含该消息ID，这样生产者就知道该消息已被正确分发。对于持久化消息，只有该消息被持久化后才返回Confirm。Confirm机制的最大优点在于异步，生产者在发送消息后即可继续其他任务。而服务器返回Confirm后，会触发生产者的回调函数，生产者在回调函数中处理Confirm信息。如果消息服务器发生异常导致消息丢失，会返回给生产者一个nack，表示消息已丢失，这样生产者就通过重发消息保证消息不丢失。Confirm机制在性能上比事务优越很多。但是Confirm机制无法回滚，一旦服务器崩溃，生产者无法得到Confirm信息，生产者其实本身也不知道该消息吃否已经被持久化，只有继续重发来保证消息不丢失，但是如果原先已经持久化的消息，并不会被回滚，这样队列中就会存在两条相同的消息，系统需要支持去重。*

* RabbitMQ的分发机制非常适合扩展，而且它是专门为并发程序设计的。如果现在load加重，那么只需要创建更多的Consumer来进行任务处理即可。  默认情况下，RabbitMQ 会顺序的分发每个Message。当每个收到ack后，会将该Message删除，然后将下一个Message分发到下一个Consumer。这种分发方式叫做round-robin。*

*ACK机制：为了保证数据不被丢失，RabbitMQ支持消息确认机制，即acknowledgments。为了保证数据能被正确处理而不仅仅是被Consumer收到，那么我们不能采用no-ack。而应该是在处理完数据后发送ack。在处理数据后发送的ack，就是告诉RabbitMQ数据已经被接收，处理完成，RabbitMQ可以去安全的删除它了。如果Consumer退出了但是没有发送ack，那么RabbitMQ就会把这个Message发送到下一个Consumer。这样就保证了在Consumer异常退出的情况下数据也不会丢失。*

*消息持久化：即使Consumer异常退出，Message也不会丢失。但是如果RabbitMQ Server退出呢？软件都有bug，即使RabbitMQ Server是完美毫无bug的（当然这是不可能的，是软件就有bug，没有bug的那不叫软件），它还是有可能退出的：被其它软件影响，或者系统重启了，系统panic了。。。为了保证在RabbitMQ退出或者crash了数据仍没有丢失，需要将queue和Message都要持久化。queue的持久化需要在声明时指定durable=True：channel.queue_declare(queue='hello', durable=True)*

*关于持久化：为了数据不丢失，我们采用：（1）数据处理结束后发ack，这样RabbitMQ Server会认为Message Deliver 成功。（2）持久化queue，可防止RabbitMQ Server 重启或crash引起的数据丢失。（3）持久化Message，理由同上。这样能保证数据100%不丢失吗？不能。问题在于RabbitMQ需要时间把这些信息存到磁盘，这个时间虽然短但的确还是有。这个时间窗口内如果数据没保存数据还会丢失。另一个原因就是RabbitMQ并不是为每个Message做fsync：它可能仅仅是把它保存到Cache，还没来得及保存到物理磁盘。*

*负载均衡：通过 basic.qos 方法设置prefetch_count=1 。这样RabbitMQ就会使每个Consumer在同一时间点最多处理一个Message。换句话说，在接收到该Consumer的ack前，不会将新Message分发给它。 设置方法如下：channel.basic_qos(prefetch_count=1)。注意，这种方法可能会导致queue满。当然，这种情况下你可能需要添加更多的Consumer，或者创建更多的virtualHost来细化你的设计。*
