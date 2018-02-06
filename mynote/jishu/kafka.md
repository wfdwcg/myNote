kafka.md

*http://www.jasongj.com/2015/03/10/KafkaColumn1/?spm=a2c4e.11153940.blogcont66676.16.6031960WXd8Te*

*Kafka集群中包含若干Producer，若干broker（Kafka支持水平扩展，一般broker数量越多，集群吞吐率越高），若干Consumer Group，以及一个Zookeeper集群。Kafka通过Zookeeper管理集群配置、选举leader，以及在Consumer Group变化时进行rebalance。Producer使用push模式将消息发布到broker，Consumer使用pull模式从broker订阅并消费消息。*

*Topic可以被认为是一个queue，每条消息都必须指定Topic，可理解为必须指明把消息放进哪个queue。为了使Kafka的吞吐率可以线性提高，把Topic分成一或多个Partition，每个Partition在物理上对应一个文件夹，该文件夹下存储这个Partition的所有消息和索引文件。若创建topic1和topic2两个topic，且分别有13和19个分区，整个集群会生成32个文件夹*

*因为每条消息都被append到该Partition中，属于顺序写磁盘，因此效率非常高（经验证，顺序写磁盘效率比随机写内存还要高，这是Kafka高吞吐率的一个很重要的保证）。*

*传统message queue一般会删除已被消费的消息，而Kafka集群会保留所有消息。Kafka提供两种策略删除旧数据。一是基于时间，二是基于Partition文件大小。例如可配置$KAFKA_HOME/config/server.properties，让Kafka删除一周前的数据，也可在Partition文件超过1GB时删除旧数据.Kafka读取特定消息的时间复杂度为O(1)，与文件大小无关，所以删除过期文件与提高Kafka性能无关。*

*Kafka会为每个Consumer Group保留一些metadata信息——当前消费消息的position，即offset。这个offset由Consumer控制。正常情况下Consumer会在消费完一条消息后递增该offset。因为offet由Consumer控制，所以Kafka broker无状态，它不需要标记哪些消息被消费过，也不需要通过broker去保证同一个Consumer Group只有一个Consumer能消费某一条消息，因此就不需要锁，这为Kafka的高吞吐率提供了有力保障。*

##消息路由
*Producer发送消息到broker时，会选择将其存储到哪一个Partition。如果Partition机制设置合理，所有消息可均匀分布到不同Partition，这样就实现了负载均衡。如果一个Topic对应一个文件，那这个文件所在的机器I/O将会成为这个Topic的性能瓶颈，而有了Partition后，不同消息可以并行写入不同broker的不同Partition里，极大提高了吞吐率。可以在$KAFKA_HOME/config/server.properties中通过配置项num.partitions来指定新建Topic的默认Partition数量，也可在创建Topic时通过参数指定。*

*发送消息时可以指定这条消息的key，Producer根据这个key和Partition机制判断应该将消息发到哪个Parition。*

*使用Consumer high level API时，同一Topic的一条消息只能被同一个Consumer Group内的一个Consumer消费，但多个Consumer Group可同时消费这一消息。*

*push模式很难适应消费速率不同的消费者，因为消息发送速率是由broker决定。push的目标是尽可能快的传递消息，但容易造成Consumer来不及处理消息，典型表现就是拒绝服务以及网络拥塞。而pull模式可以根据Consumer消费能力以适当速率消费。对于Kafka而言pull模式更合适。pull模式可简化broker设计，Consumer可自主控制消费消息速率，同时Consumer可以自己控制消费方式——即可批量消费也可逐条消费，同时还能选择不同的提交方式从而实现不同的传输语义。*

*有这么几种可能的delivery guarantee：
At most once 消息可能会丢，但绝不会重复传输
At least one 消息绝不会丢，但可能会重复传输
Exactly once 每条消息肯定会被传输一次且仅传输一次，很多时候这是用户所想要的。*

*读完消息先commit再处理。这种模式下如果Consumer在commit后还没处理消息就crash，重新工作后就无法读到刚刚已提交而未处理的消息，这对应于At most once*

*读完消息先处理再commit。这种模式下如果处理消息后commit前Consumer crash，重新工作时还会处理未commit的消息，实际上该消息已被处理。这对应于At least once。在很多使用场景下，消息都有一个主键，所以消息处理往往具有幂等性，即多次处理这条消息跟只处理一次是等效的，那就可以认为是Exactly once。（这种说法比较牵强，毕竟它不是Kafka本身提供的机制，主键本身也并不能完全保证操作幂等。我们不应把处理过程的特性——如是否幂等性，当成Kafka的Feature）*

*如果一定要Exactly once，就需要协调offset和实际操作的输出。经典做法是引入两阶段提交。如果能让offset和操作输入存在同一地方，会更简洁和通用。这种方式可能更好，因为许多输出系统可能不支持两阶段提交。比如，Consumer拿到数据后可能把数据放到HDFS，如果把最新的offset和数据本身一起写到HDFS，那就可以保证数据的输出和offset的更新要么都完成，要么都不完成，间接实现Exactly once。（目前就high level API而言，offset是存于Zookeeper中的，无法存于HDFS，而low level API的offset是由自己去维护的，可以将之存于HDFS中）.总之，Kafka默认保证At least once，并且允许通过设置Producer异步提交来实现At most once。而Exactly once要求与外部存储系统协作，幸运的是Kafka提供的offset可以非常直接非常容易得使用这种方式。*

###高可用
*在Kafka0.8以前没有Replication，某个Broker宕机则其上所有Partition数据都不可被消费，这与Kafka数据持久性及Delivery Guarantee设计目标相悖。同时Producer都不能再将数据存于这些Partition中。*

*引入Replication之后，同一个Partition可能会有多个Replica，而这时需要在这些Replica中选出一个Leader，Producer和Consumer只与这个Leader交互，其它Replica作为Follower从Leader中复制数据。因为需要保证同一个Partition的多个Replica之间的数据一致性（其中一个宕机后其它Replica必须要能继续服务并且即不能造成数据重复也不能造成数据丢失）。如果没有一个Leader，所有Replica都可同时读/写数据，那就需要保证多个Replica之间互相（N×N条通路）同步数据，数据的一致性和有序性非常难保证，大大增加了Replication实现的复杂性，同时也增加了出现异常的几率。而引入Leader后，只有Leader负责数据读写，Follower只向Leader顺序Fetch数据（N条通路），系统更加简单且高效。*

*为了更好的做负载均衡，Kafka尽量将所有的Partition均匀分配到整个集群上。一个典型的部署方式是一个Topic的Partition数量大于Broker的数量。同时为了提高Kafka的容错能力，也需要将同一个Partition的Replica尽量分散到不同的机器。实际上，如果所有的Replica都在同一个Broker上，那一旦该Broker宕机，该Partition的所有Replica都无法工作，也就达不到HA的效果。同时，如果某个Broker宕机了，需要保证它上面的负载可以被均匀的分配到其它幸存的所有Broker上。*

*Producer在发布消息到某个Partition时，先通过Zookeeper找到该Partition的Leader，然后无论该Topic的Replication Factor为多少（也即该Partition有多少个Replica），Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向Producer发送ACK。为了提高性能，每个Follower在接收到数据后就立马向Leader发送ACK，而非等到数据写入Log中。因此，对于已经commit的消息，Kafka只能保证它被存于多个Replica的内存中，而不能保证它们被持久化到磁盘中，也就不能完全保证异常发生后该条消息一定能被Consumer消费。但考虑到这种场景非常少见，可以认为这种方式在性能和数据持久化上做了一个比较好的平衡。在将来的版本中，Kafka会考虑提供更高的持久性。Consumer读消息也是从Leader读取，只有被commit过的消息（offset低于HW的消息）才会暴露给Consumer。*

*Leader会跟踪与其保持同步的Replica列表，该列表称为ISR（即in-sync Replica）。如果一个Follower宕机，或者落后太多，Leader将把它从ISR中移除。这里所描述的“落后太多”指Follower复制的消息落后于Leader后的条数超过预定值（该值可在$KAFKA_HOME/config/server.properties中通过replica.lag.max.messages配置，其默认值是4000）或者Follower超过一定时间（该值可在$KAFKA_HOME/config/server.properties中通过replica.lag.time.max.ms来配置，其默认值是10000）未向Leader发送fetch请求。。*

*Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的Follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率（高吞吐率是Kafka非常重要的一个特性）。而异步复制方式下，Follower异步的从Leader复制数据，数据只要被Leader写入log就被认为已经commit，这种情况下如果Follower都复制完都落后于Leader，而如果Leader突然宕机，则会丢失数据。而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。Follower可以批量的从Leader复制数据，这样极大的提高复制性能（批量写磁盘），极大减少了Follower与Leader的差距。*

*需要说明的是，Kafka只解决fail/recover，不处理“Byzantine”（“拜占庭”）问题。一条消息只有被ISR里的所有Follower都从Leader复制过去才会被认为已提交。这样就避免了部分数据被写进了Leader，还没来得及被任何Follower复制就宕机了，而造成数据丢失（Consumer无法消费这些数据）。而对于Producer而言，它可以选择是否等待消息commit，这可以通过request.required.acks来设置。这种机制确保了只要ISR有一个或以上的Follower，一条被commit的消息就不会丢失。*

*一个很重要的问题是当Leader宕机了，怎样在Follower中选举出新的Leader。因为Follower可能落后许多或者crash了，所以必须确保选择“最新”的Follower作为新的Leader。一个基本的原则就是，如果Leader不在了，新的Leader必须拥有原来的Leader commit过的所有消息。这就需要作一个折衷，如果Leader在标明一条消息被commit前等待更多的Follower确认，那在它宕机之后就有更多的Follower可以作为新的Leader，但这也会造成吞吐率的下降。*

*一种非常常用的Leader Election的方式是“Majority Vote”（“少数服从多数”），但Kafka并未采用这种方式。这种模式下，如果我们有2f+1个Replica（包含Leader和Follower），那在commit之前必须保证有f+1个Replica复制完消息，为了保证正确选出新的Leader，fail的Replica不能超过f个。因为在剩下的任意f+1个Replica里，至少有一个Replica包含有最新的所有消息。这种方式有个很大的优势，系统的latency只取决于最快的几个Broker，而非最慢那个。Majority Vote也有一些劣势，为了保证Leader Election的正常进行，它所能容忍的fail的follower个数比较少。如果要容忍1个follower挂掉，必须要有3个以上的Replica，如果要容忍2个Follower挂掉，必须要有5个以上的Replica。也就是说，在生产环境下为了保证较高的容错程度，必须要有大量的Replica，而大量的Replica又会在大数据量下导致性能的急剧下降。这就是这种算法更多用在Zookeeper这种共享集群配置的系统中而很少在需要存储大量数据的系统中使用的原因。例如HDFS的HA Feature是基于majority-vote-based journal，但是它的数据存储并没有使用这种方式。*

*实际上，Leader Election算法非常多，比如Zookeeper的Zab, Raft和Viewstamped Replication。而Kafka所使用的Leader Election算法更像微软的PacificA算法。*

*Kafka在Zookeeper中动态维护了一个ISR（in-sync replicas），这个ISR里的所有Replica都跟上了leader，只有ISR里的成员才有被选为Leader的可能。在这种模式下，对于f+1个Replica，一个Partition能在保证不丢失已经commit的消息的前提下容忍f个Replica的失败。在大多数使用场景中，这种模式是非常有利的。事实上，为了容忍f个Replica的失败，Majority Vote和ISR在commit前需要等待的Replica数量是一样的，但是ISR需要的总的Replica的个数几乎是Majority Vote的一半。*

*上文提到，在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某个Partition的所有Replica都宕机了，就无法保证数据不丢失了。这种情况下有两种可行的方案：（1）等待ISR中的任一个Replica“活”过来，并且选它作为Leader；（2）选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader*


####如何选举Leader
*最简单最直观的方案是，所有Follower都在Zookeeper上设置一个Watch，一旦Leader宕机，其对应的ephemeral znode会自动删除，此时所有Follower都尝试创建该节点，而创建成功者（Zookeeper保证只有一个能创建成功）即是新的Leader，其它Replica即为Follower。*

*Kafka保证同一Consumer Group中只有一个Consumer会消费某条消息，实际上，Kafka保证的是稳定状态下每一个Consumer实例只会消费某一个或多个特定Partition的数据，而某个Partition的数据只会被某一个特定的Consumer实例所消费。也就是说Kafka对消息的分配是以Partition为单位分配的，而非以每一条消息作为分配单元。这样设计的劣势是无法保证同一个Consumer Group里的Consumer均匀消费数据，优势是每个Consumer不用都跟大量的Broker通信，减少通信开销，同时也降低了分配难度，实现也更简单。另外，因为同一个Partition里的数据是有序的，这种设计可以保证每个Partition里的数据可以被有序消费。*

*单分区问题：如果某Consumer Group中Consumer（每个Consumer只创建1个MessageStream）数量少于Partition数量，则至少有一个Consumer会消费多个Partition的数据，如果Consumer的数量与Partition数量相同，则正好一个Consumer消费一个Partition的数据。而如果Consumer的数量多于Partition的数量时，会有部分Consumer无法消费该Topic下任何一条消息。*