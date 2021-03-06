### 过期部分

数据库如何保存键的生存时间和过期时间，与及服务器如何自动删除那些带有生存时间和过期时间的键两个问题。



redisDb结构的expires字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典：**过期字典的键是一个指针**，这个指针指向键空间中的某个键对象（也即是某个数据库键）。



**设置过期时间时，服务器会在数据库的过期字典中关联给定的数据库键和过期时间。**



### 持久化部分

与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的。



redis的服务器进程就是一个事件循环，这个循环中的文件事件负责接收客户端的命令请求，与及向客户端发送命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数。



刷盘配置：

always，每次循环后都将aof_buf缓冲区中的所有内容写入到AOF文件中，并且同步

everysec,每隔1秒才同步

no，根据操作系统决定同步



### 槽迁移部分

Redis 迁移的单位是槽，Redis 一个槽一个槽进行迁移，当一个槽正在迁移时，这个槽就处于中间过渡状态。**这个槽在原节点的状态为`migrating`，在目标节点的状态为`importing`，表示数据正在从源流向目标。**



首先新旧两个节点对应的槽位都存在部分 key 数据。客户端先尝试访问旧节点，如果对应的数据还在旧节点里面，那么旧节点正常处理。如果对应的数据不在旧节点里面，那么有两种可能，要么该数据在新节点里，要么根本就不存在。旧节点不知道是哪种情况，所以它会向客户端返回一个-ASK targetNodeAddr的重定向指令。客户端收到这个重定向指令后，先去目标节点执行一个不带任何参数的asking指令，然后在目标节点再重新执行原先的操作指令。.



### Gossip协议

在分布式存储中需要提供维护节点元数据信息的机制，元数据是指：节点负责哪些数据，是否出现故障等状态信息。常见的元数据维护方式为：集中式和p2p式。



Gossip协议的工作原理就是节点彼此不断通信交换信息。常用的Gossip消息可分为：ping消息、pong消息、meet消息、fail消息。**meet消息用于新节点加入，收到meet消息且对方是新节点，那就开始握手通信。**集群内每个节点每秒向多个其他节点发送ping消息，用于检测节点是否在线和交换彼此状态信息。ping消息发送封装了自身节点和部分其他节点的状态数据。（**最频繁的消息类型，而且消息会携带当前节点和部分其他节点的状态数据，如果通信节点选择过多就会加重带宽和计算成本，如果通信节点选择过少就会降低集群内所有节点彼此信息交换频率，从而影响故障判定或者新节点发现速度**，redis集群每个节点维护个定时任务默认每秒执行10次，每秒会随机选取5个节点找出最久没有通信的节点发送ping消息）。pong消息除了作为ping、meet消息的响应消息之外，节点可以向集群内广播自身的pong消息来通知整个集群对自身状态进行更新。当节点判定集群内另一个节点下线时，会向集群内广播一个fail消息。



### 故障转移

主观下线（正常ping/pong后会更新最后的通信时间）

客观下线（半数以上的节点认定才算）