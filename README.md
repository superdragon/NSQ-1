
# 分布式消息处理平台

## 这是**go-NSQ**的工程应用

NSQ是一个实时的分布式消息平台。它的设计目标是为在多台计算机上运行的松散服务提供一个现代化的基础设施骨架。

- NSQ 设计的目的是用来大规模地处理每天数以十亿计级别的消息。

- NSQ 具有分布式和去中心化拓扑结构，该结构具有无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。

- NSQ 非常容易配置和部署，且具有最大的灵活性，支持众多消息协议。另外，官方还提供了拆箱即用 Go 和 Python 库。


## NSQ的自身特色很明显，最主要的优势在如下三个方面：

- 1，性能。在多个著名网站生产环境中被采用，每天能够处理数亿级别的消息。参见官方提供的性能说明文档

- 2，易用性。非常易于部署（几乎没有依赖）和配置（所有参数都可以通过命令行进行配置）。

- 3，可扩展性。具有分布式且无单点故障的拓扑结构，支持水平扩展，在无中断情况下能够无缝地添加集群节点。还具有强大的集群管理界面，参见nsqadmin

## NSQ是由3个进程组成的：
- nsqd是一个接收、排队、然后转发消息到客户端的进程。
- nsqlookupd 管理拓扑信息并提供最终一致性的发现服务。
- nsqadmin用于实时查看集群的统计数据（并且执行各种各样的管理任务）。

NSQ中的数据流模型是由streams和consumers组成的tree。topic是一种独特的stream。channel是一个订阅了给定topic consumers 逻辑分组。
 
![](/images/nsq-1.png?raw=true)

**Topics 和 Channels**

Topics 和 channels，是NSQ的核心成员，它们是如何使用go语言的特点来设计系统的最好示例。

Go的channels（为防止歧义，以下简称为“go-chan”）是表达队列的一种自然方式，因此一个NSQ的topic/channel，其核心就是一个存放消息指针的go-chan缓冲区。缓冲区的大小由  --mem-queue-size 配置参数确定。

读取数据后，向topic发布消息的行为包括：
- 实例化消息结构 (并分配消息体的字节数组)
- read-lock 并获得 Topic
- read-lock 并检查是否可以发布
- 发送到go-chan缓冲区

为了从一个topic和它的channels获得消息，topic不能按典型的方式用go-chan来接收，因为多个goroutines在一个go-chan上接收将会分发消息，而期望的结果是把每个消息复制到所有channel(goroutine)中。

此外，每个topic维护3个主要goroutine。第一个叫做 router，负责从传入的go-chan中读取新发布的消息，并存储到一个队列里（内存或硬盘）。

第二个，称为 messagePump, 它负责复制和推送消息到如上所述的channel中。

第三个负责 DiskQueue IO，将在后面讨论。

Channels稍微有点复杂，它的根本目的是向外暴露一个单输入单输出的go-chan（事实上从抽象的角度来说，消息可能存在内存里或硬盘上）；
![enter description here][2]

另外，每一个channel维护2个时间优先级队列，用于延时和消息超时的处理（并有2个伴随goroutine来监视它们）。

并行化的改善是通过管理每个channel的数据结构来实现，而不是依靠go运行时的全局定时器。

注意：在内部，go运行时使用一个优先级队列和goroutine来管理定时器。它为整个time包（但不局限于）提供了支持。它通常不需要用户来管理时间优先级队列，但一定要记住，它是一个有锁的数据结构，有可能会影响 GOMAXPROCS>1 的性能。请参阅runtime/time.goc。

**Backend / DiskQueue**

NSQ的一个设计目标是绑定内存中的消息数目。它是通过DiskQueue(它拥有前面提到的的topic或channel的第三个goroutine)透明的把消息写入到磁盘上来实现的。

由于内存队列只是一个go-chan，没必要先把消息放到内存里，如果可能的话，退回到磁盘上：

``` go
for msg := range c.incomingMsgChan {
    select {
    case c.memoryMsgChan <- msg:
    default:
        err := WriteMessageToBackend(&msgBuf, msg, c.backend)
        if err != nil {
            // ... handle errors ...
        }
    }
}
```
利用go语言的select语句，只需要几行代码就可以实现这个功能：上面的default分支只有在memoryMsgChan 满的情况下才会执行。

NSQ也有临时channel的概念。临时channel会丢弃溢出的消息（而不是写入到磁盘），当没有客户订阅后它就会消失。这是一个Go接口的完美用例。Topics和channels有一个的结构成员被声明为Backend接口，而不是一个具体的类型。一般的 topics和channels使用DiskQueue，而临时channel则使用了实现Backend接口的DummyBackendQueue。

## NSQ分布式实时消息平台用法

NSQ是一个基于Go语言的分布式实时消息平台，可用于大规模系统中的实时消息服务，并且每天能够处理数亿级别的消息，其设计目标是为在分布式环境下运行的去中心化服务提供一个强大的基础架构。

NSQ非常容易配置和部署，且具有最大的灵活性，支持众多消息协议。

### NSQ是由四个重要组件构成：

- nsqd：一个负责接收、排队、转发消息到客户端的守护进程，它可以独立运行，不过通常它是由 nsqlookupd 实例所在集群配置的
- nsqlookupd：管理拓扑信息并提供最终一致性的发现服务的守护进程
- nsqadmin：一套Web用户界面，可实时查看集群的统计数据和执行各种各样的管理任务
- utilities：常见基础功能、数据流处理工具，如nsq_stat、nsq_tail、nsq_to_file、nsq_to_http、nsq_to_nsq、to_nsq

## 快速启动NSQ

```bash
brew install nsq
```

启动拓扑发现 

```bash
nsqlookupd
```

启动主服务、并注册 

```bash
nsqd --lookupd-tcp-address=127.0.0.1:4160
```

启动WEB UI管理程序 

```bash
nsqadmin --lookupd-http-address=127.0.0.1:4161
```

## 快速启动NSQ

brew install nsq

启动拓扑发现 nsqlookupd

启动主服务、并注册 nsqd --lookupd-tcp-address=127.0.0.1:4160

启动WEB UI管理程序 nsqadmin --lookupd-http-address=127.0.0.1:4161

## 简单使用演示

可以用浏览器访问http://127.0.0.1:4171/观察数据

也可尝试下 watch -n 0.5 "curl -s http://127.0.0.1:4151/stats" 监控统计数据

发布一个消息 

```bash
curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'
```

创建一个消费者 

```bash
nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161
```

### Golang使用NSQ

go-nsq Golang客户端库（官方客户端开发库）。测试实例如下：


```go
package main

import (
    "fmt"
    "time"

    "github.com/nsqio/go-nsq"
)

// ConsumerHandler 消费者处理者
type ConsumerHandler struct{}

// HandleMessage 处理消息
func (*ConsumerHandler) HandleMessage(msg *nsq.Message) error {
    fmt.Println(string(msg.Body))
    return nil
}

// Producer 生产者
func Producer() {
    producer, err := nsq.NewProducer("127.0.0.1:4150", nsq.NewConfig())
    if err != nil {
        fmt.Println("NewProducer", err)
        panic(err)
    }

    i := 1
    for {
        if err := producer.Publish("test", []byte(fmt.Sprintf("Hello World %d", i))); err != nil {
            fmt.Println("Publish", err)
            panic(err)
        }

        time.Sleep(time.Second * 5)

        i++
    }
}

// ConsumerA 消费者
func ConsumerA() {
    consumer, err := nsq.NewConsumer("test", "test-channel-a", nsq.NewConfig())
    if err != nil {
        fmt.Println("NewConsumer", err)
        panic(err)
    }

    consumer.AddHandler(&ConsumerHandler{})

    if err := consumer.ConnectToNSQLookupd("127.0.0.1:4161"); err != nil {
        fmt.Println("ConnectToNSQLookupd", err)
        panic(err)
    }
}

// ConsumerB 消费者
func ConsumerB() {
    consumer, err := nsq.NewConsumer("test", "test-channel-b", nsq.NewConfig())
    if err != nil {
        fmt.Println("NewConsumer", err)
        panic(err)
    }

    consumer.AddHandler(&ConsumerHandler{})

    if err := consumer.ConnectToNSQLookupd("127.0.0.1:4161"); err != nil {
        fmt.Println("ConnectToNSQLookupd", err)
        panic(err)
    }
}

func main() {
    ConsumerA()
    ConsumerB()
    Producer()
}
```
命令执行顺序如下
```bash
nsqlookupd
nsqd --lookupd-tcp-address=127.0.0.1:4160 --broadcast-address=127.0.0.1
nsqadmin --lookupd-http-address=127.0.0.1:4161
```












## 介绍
NSQ是一个基于Go语言的分布式实时消息平台，它基于MIT开源协议发布，由bitly公司开源出来的一款简单易用的消息中间件。

官方和第三方还为NSQ开发了众多客户端功能库，如官方提供的基于HTTP的nsqd、Go客户端go-nsq、Python客户端pynsq、基于Node.js的JavaScript客户端nsqjs、异步C客户端libnsq、Java客户端nsq-java以及基于各种语言的众多第三方客户端功能库。

## Features

**1.Distributed** 

NSQ提供了分布式的，去中心化，且没有单点故障的拓扑结构，稳定的消息传输发布保障，能够具有高容错和HA（高可用）特性。

**2.Scalable易于扩展** 

NSQ支持水平扩展，没有中心化的brokers。内置的发现服务简化了在集群中增加节点。同时支持pub-sub和load-balanced 的消息分发。

**3.Ops Friendly** 

NSQ非常容易配置和部署，生来就绑定了一个管理界面。二进制包没有运行时依赖。官方有Docker image。

**4.Integrated高度集成** 

官方的 Go 和 Python库都有提供。而且为大多数语言提供了库。

## 组件

- Topic ：一个topic就是程序发布消息的一个逻辑键，当程序第一次发布消息时就会创建topic。
- Channels ：channel与消费者相关，是消费者之间的负载均衡，channel在某种意义上来说是一个“队列”。每当一个发布者发送一条消息到一个topic，消息会被复制到所有消费者连接的channel上，消费者通过这个特殊的channel读取消息，实际上，在消费者第一次订阅时就会创建channel。Channel会将消息进行排列，如果没有消费者读取消息，消息首先会在内存中排队，当量太大时就会被保存到磁盘中。
- Messages：消息构成了我们数据流的中坚力量，消费者可以选择结束消息，表明它们正在被正常处理，或者重新将他们排队待到后面再进行处理。每个消息包含传递尝试的次数，当消息传递超过一定的阀值次数时，我们应该放弃这些消息，或者作为额外消息进行处理。
- nsqd：nsqd 是一个守护进程，负责接收，排队，投递消息给客户端。它可以独立运行，不过通常它是由 nsqlookupd 实例所在集群配置的（它在这能声明 topics 和 channels，以便大家能找到）。
- nsqlookupd：nsqlookupd 是守护进程负责管理拓扑信息。客户端通过查询 nsqlookupd 来发现指定话题（topic）的生产者，并且 nsqd 节点广播话题（topic）和通道（channel）信息。有两个接口：TCP 接口，nsqd 用它来广播。HTTP 接口，客户端用它来发现和管理。
- nsqadmin：nsqadmin 是一套 WEB UI，用来汇集集群的实时统计，并执行不同的管理任务。

常用工具类：
- nsq_to_file：消费指定的话题（topic）/通道（channel），并写到文件中，有选择的滚动和/或压缩文件。
- nsq_to_http：消费指定的话题（topic）/通道（channel）和执行 HTTP requests (GET/POST) 到指定的端点。
- nsq_to_nsq：消费者指定的话题/通道和重发布消息到目的地 nsqd 通过 TCP。

## 拓扑结构

NSQ推荐通过他们相应的nsqd实例使用协同定位发布者，这意味着即使面对网络分区，消息也会被保存在本地，直到它们被一个消费者读取。更重要的是，发布者不必去发现其他的nsqd节点，他们总是可以向本地实例发布消息。

![](images/1.png?raw=true)

首先，一个发布者向它的本地nsqd发送消息，要做到这点，首先要先打开一个连接，然后发送一个包含topic和消息主体的发布命令，在这种情况下，我们将消息发布到事件topic上以分散到我们不同的worker中。

事件topic会复制这些消息并且在每一个连接topic的channel上进行排队，在我们的案例中，有三个channel，它们其中之一作为档案channel。消费者会获取这些消息并且上传到S3。

![](images/2.png?raw=true)

每个channel的消息都会进行排队，直到一个worker把他们消费，如果此队列超出了内存限制，消息将会被写入到磁盘中。Nsqd节点首先会向nsqlookup广播他们的位置信息，一旦它们注册成功，worker将会从nsqlookup服务器节点上发现所有包含事件topic的nsqd节点。

![](images/3.png?raw=true)

然后每个worker向每个nsqd主机进行订阅操作，用于表明worker已经准备好接受消息了。这里我们不需要一个完整的连通图，但我们必须要保证每个单独的nsqd实例拥有足够的消费者去消费它们的消息，否则channel会被队列堆着。




## 原理

**1.消息传递担保** 

NSQ 保证消息将交付至少一次，虽然消息可能是重复的。消费者应该关注到这一点，删除重复数据或执行idempotent等操作。
这个担保是作为协议和工作流的一部分，工作原理如下（假设客户端成功连接并订阅一个话题）：
1）客户表示已经准备好接收消息
2）NSQ 发送一条消息，并暂时将数据存储在本地（在 re-queue 或 timeout）
3）客户端回复 FIN（结束）或 REQ（重新排队）分别指示成功或失败。如果客户端没有回复, NSQ 会在设定的时间超时，自动重新排队消息
这确保了消息丢失唯一可能的情况是不正常结束 nsqd 进程。在这种情况下，这是在内存中的任何信息（或任何缓冲未刷新到磁盘）都将丢失。
如何防止消息丢失是最重要的，即使是这个意外情况可以得到缓解。一种解决方案是构成冗余 nsqd对（在不同的主机上）接收消息的相同部分的副本。因为你实现的消费者是幂等的，以两倍时间处理这些消息不会对下游造成影响，并使得系统能够承受任何单一节点故障而不会丢失信息。

**2.简化配置和管理** 

单个 nsqd 实例被设计成可以同时处理多个数据流。流被称为“话题”和话题有 1 个或多个“通道”。每个通道都接收到一个话题中所有消息的拷贝。在实践中，一个通道映射到下行服务消费一个话题。

话题和通道都没有预先配置。话题由第一次发布消息到命名的话题或第一次通过订阅一个命名话题来创建。通道被第一次订阅到指定的通道创建。话题和通道的所有缓冲的数据相互独立，防止缓慢消费者造成对其他通道的积压（同样适用于话题级别）。

一个通道一般会有多个客户端连接。假设所有已连接的客户端处于准备接收消息的状态，每个消息将被传递到一个随机的客户端。nsqlookupd，它提供了一个目录服务，消费者可以查找到提供他们感兴趣订阅话题的 nsqd 地址 。在配置方面，把消费者与生产者解耦开（它们都分别只需要知道哪里去连接 nsqlookupd 的共同实例，而不是对方），降低复杂性和维护。

在更底的层面，每个 nsqd 有一个与 nsqlookupd 的长期 TCP 连接，定期推动其状态。这个数据被 nsqlookupd 用于给消费者通知 nsqd 地址。对于消费者来说，一个暴露的 HTTP /lookup 接口用于轮询。为话题引入一个新的消费者，只需启动一个配置了 nsqlookup 实例地址的 NSQ 客户端。无需为添加任何新的消费者或生产者更改配置，大大降低了开销和复杂性。

**3.消除单点故障** 

NSQ被设计以分布的方式被使用。nsqd 客户端（通过 TCP ）连接到指定话题的所有生产者实例。没有中间人，没有消息代理，也没有单点故障。

这种拓扑结构消除单链，聚合，反馈。相反，你的消费者直接访问所有生产者。从技术上讲，哪个客户端连接到哪个 NSQ 不重要，只要有足够的消费者连接到所有生产者，以满足大量的消息，保证所有东西最终将被处理。对于 nsqlookupd，高可用性是通过运行多个实例来实现。他们不直接相互通信和数据被认为是最终一致。消费者轮询所有的配置的 nsqlookupd 实例和合并 response。失败的，无法访问的，或以其他方式故障的节点不会让系统陷于停顿。

**4.效率** 

对于数据的协议，通过推送数据到客户端最大限度地提高性能和吞吐量的，而不是等待客户端拉数据。这个概念，称之为 RDY 状态，基本上是客户端流量控制的一种形式。

当客户端连接到 nsqd 和并订阅到一个通道时，它被放置在一个 RDY 为 0 状态。这意味着，还没有信息被发送到客户端。当客户端已准备好接收消息发送，更新它的命令 RDY 状态到它准备处理的数量，比如 100。无需任何额外的指令，当 100 条消息可用时，将被传递到客户端（服务器端为那个客户端每次递减 RDY 计数）。客户端库的被设计成在 RDY 数达到配置 max-in-flight的 25% 发送一个命令来更新 RDY 计数（并适当考虑连接到多个 nsqd 情况下，适当地分配）。
![](images/4.png?raw=true)

**5.心跳和超时** 

NSQ 的 TCP 协议是面向 push 的。在建立连接，握手，和订阅后，消费者被放置在一个为 0 的 RDY 状态。当消费者准备好接收消息，它更新的 RDY 状态到准备接收消息的数量。NSQ 客户端库不断在幕后管理，消息控制流的结果。每隔一段时间，nsqd 将发送一个心跳线连接。客户端可以配置心跳之间的间隔，但 nsqd 会期待一个回应在它发送下一个心掉之前。
组合应用级别的心跳和 RDY 状态，避免头阻塞现象，也可能使心跳无用（即，如果消费者是在后面的处理消息流的接收缓冲区中，操作系统将被填满，堵心跳）为了保证进度，所有的网络 IO 时间上限势必与配置的心跳间隔相关联。这意味着，你可以从字面上拔掉之间的网络连接 nsqd 和消费者，它会检测并正确处理错误。当检测到一个致命错误，客户端连接被强制关闭。在传输中的消息会超时而重新排队等待传递到另一个消费者。最后，错误会被记录并累计到各种内部指标。

**6.分布式** 

因为NSQ没有在守护程序之间共享信息，所以它从一开始就是为了分布式操作而生。个别的机器可以随便宕机随便启动而不会影响到系统的其余部分，消息发布者可以在本地发布，即使面对网络分区。
这种“分布式优先”的设计理念意味着NSQ基本上可以永远不断地扩展，需要更高的吞吐量？那就添加更多的nsqd吧。唯一的共享状态就是保存在lookup节点上，甚至它们不需要全局视图，配置某些nsqd注册到某些lookup节点上这是很简单的配置，唯一关键的地方就是消费者可以通过lookup节点获取所有完整的节点集。清晰的故障事件——NSQ在组件内建立了一套明确关于可能导致故障的的故障权衡机制，这对消息传递和恢复都有意义。虽然它们可能不像Kafka系统那样提供严格的保证级别，但NSQ简单的操作使故障情况非常明显。

**7.no replication** 

不像其他的队列组件，NSQ并没有提供任何形式的复制和集群，也正是这点让它能够如此简单地运行，但它确实对于一些高保证性高可靠性的消息发布没有足够的保证。我们可以通过降低文件同步的时间来部分避免，只需通过一个标志配置，通过EBS支持我们的队列。但是这样仍然存在一个消息被发布后马上死亡，丢失了有效的写入的情况。

**8.没有严格的顺序** 
 
虽然Kafka由一个有序的日志构成，但NSQ不是。消息可以在任何时间以任何顺序进入队列。在我们使用的案例中，这通常没有关系，因为所有的数据都被加上了时间戳，但它并不适合需要严格顺序的情况。

**9.无数据重复删除功能**
 
NSQ对于超时系统，它使用了心跳检测机制去测试消费者是否存活还是死亡。很多原因会导致我们的consumer无法完成心跳检测，所以在consumer中必须有一个单独的步骤确保幂等性。


## nsqadmin
对Streams的详细信息进行查看，包括NSQD节点，具体的channel，队列中的消息数，连接数等信息。

![](images/5.png?raw=true)

![](images/6.png?raw=true)

列出所有的NSQD节点

![](images/7.png?raw=true)

消息的统计

![](images/8.png?raw=true)

lookup主机的列表

![](images/9.png?raw=true)
 
## 实践
拓扑结构

![](images/10.png?raw=true)

实验采用3台NSQD服务，2台LOOKUPD服务。采用官方推荐的拓扑，消息发布的服务和NSQD在一台主机。一共5台机器。NSQ基本没有配置文件，配置通过命令行指定参数。主要命令如下:
- LOOKUPD命令：

```bash
bin/nsqlookupd
```

- NSQD命令：

```bash
bin/nsqd --lookupd-tcp-address=172.16.30.254:4160 -broadcast-address=172.16.30.254
bin/nsqadmin --lookupd-http-address=172.16.30.254:4161
```

- 工具类，消费后存储到本地文件。

```bash
bin/nsq_to_file --topic=newtest --channel=test --output-dir=/tmp --lookupd-http-address=172.16.30.254:4161
```

- 发布一条消息

```bash
curl -d 'hello world 5' 'http://172.16.30.254:4151/put?topic=test'
```

## 总结
NSQ基本核心就是简单性，是一个简单的队列，这意味着它很容易进行故障推理和很容易发现bug。消费者可以自行处理故障事件而不会影响系统剩下的其余部分。

事实上，简单性是我们决定使用NSQ的首要因素，这方便与我们的许多其他软件一起维护，通过引入队列使我们得到了堪称完美的表现，通过队列甚至让我们增加了几个数量级的吞吐量。越来越多的consumer需要一套严格可靠性和顺序性保障，这已经超过了NSQ提供的简单功能。

结合我们的业务系统来看，对于我们所需要传输的发票消息，相对比较敏感，无法容忍某个nsqd宕机，或者磁盘无法使用的情况，该节点堆积的消息无法找回。这是我们没有选择该消息中间件的主要原因。简单性和可靠性似乎并不能完全满足。相比Kafka，ops肩负起更多负责的运营。另一方面，它拥有一个可复制的、有序的日志可以提供给我们更好的服务。但对于其他适合NSQ的consumer，它为我们服务的相当好，我们期待着继续巩固它的坚实的基础。




## 参考
http://www.infoq.com/cn/news/2015/02/nsq-distributed-message-platform/

https://speakerdeck.com/snakes/nsq-nyc-golang-meetup

http://nsq.io/









