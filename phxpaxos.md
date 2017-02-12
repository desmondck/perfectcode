# PhxPaxos架构设计、实现分析

## 架构设计

Paxos为分布式一致性协议，用于解决异步通信环境下分布式系统的一致性问题。做为分布式系统，必定存在多个节点共同参与\(一般来说节点数量为奇数个，常用的节点数量如3，5等\)。

一个Paxos实例执行可以确定一个值，其最实际的工程价值莫过于提供选主服务。但如果可以有序的确定多个值，再结合状态机将值赋予业务含义\(如日志回放等\)，Paxos将具有更大的价值。

PhxPaxos基于《Paxos made simple》实现，可以做到有序确定多个值，更进一步的是其可以同时确定多组有序的值。

PhxPaxos整体架构如下：

* 一个Group用于有序的确定多个值，多个Group可以同时确定多组有序的值。
* Group中的每个Node代表一个物理节点\(VM或进程\)，一个Node中同时运行了多个Group的Instance，如下表

|  | Group 0 | Group 1 | Group 2 |
| :--- | :--- | :--- | :--- |
| Node A | Instance X | Instance Y | Instance Z |
| Node B | Instance X | Instance Y | Instance Z |
| Node C | Instance X | Instance Y | Instance Z |

* 每个Instance为Paxos协议的执行主体，用于确定一个值，当Instance N的值被确定后，Instance N被销毁，构建Instance N+1确定下一个值。

#### 图1

### Node

一个Node对应一个物理节点，如果需要正确的运行Paxos协议，必须由以下部分组成：

* **网络模块**
  * 负责节点间网络通信。

* **数据存储**
  * 负责记录确定的有序值。
* **Paxos分组**
  * 上一节提到的一个Node同时运行多个Group的能力。
  * 在实现上，Node维护了一个Group的列表，每个Group中包含一个Instance对象
  * instance负责掌控整个Paxos协议的运行，也包括对其他机制的调控。
* **选主服务**
  * 用于确定Paxos的主节点。

> 主节点不是必须的，但选主后仅由主节点发起服务可以有效的提升性能。
>
> 选主服务有两点比较特殊：
>
> 1. 本身使用了Paxos协议
> 2. 每个Group需要单独选主\(存在另外一种实现：选主服务下放到Paxos分组中去完成\)

PhxPaxos中的节点除了做为Paxos协议的参与者，还运行另外一类成为follower的节点。Follower指定一个运行Paxos协议的节点用于数据同步，它节点不参与Paxos协议，也不参与Paxos选主。Follower更像传统意义上的同步备，当Paxos协议节点确定一个值后，将数据同步到Follower节点。但有一点不同的是：Follower节点运行Learner，当某个值缺失时，可以通过Learner主动发起AskForLearn习得。

图2

### Instance

每个Node的每个Group下运行一个Instance实例，Instance实例用于确定当前值。多个Node相同Group中的Instance相互通信，完成Paxos协议.

Instance由以下部分组成：

* Proposer
  * Paxos提案发起者。
* Acceptor
  * Paxos提案接收者。
* Learner
  * Paxos值习得者。

CheckpointMgr

* 镜像数据管理者。
* 指引业务生成镜像数据，一旦指定instance id之前的镜像数据产生，理论上就可以移除该instance id之前的Paxos Log数据，以免空间的无限扩展。这部分可以参加《微信自研生产级paxos类库PhxPaxos实现原理介绍》、《状态机Checkpoint详解》。

图3

## 实现分析

### 网络实现

PhxPaxos中网络接口抽象为NetWork，内置实现类为DFNetWrok，支持UDP、TCP两种协议。PhxPaxos将网络操作全异步化，一共启了5个线程：

* **UDPRecv**
  * 创建一个基于UDP协议的socket对象
  * 线程负责接收来自网络其他节点的UDP消息，并通知Instance处理\(根据消息中的group id找到正确的group\)
* **UDPSend**
  * 所有Instance需要发送的UDP消息预先放入消息队列
  * 线程负责将消息队列中的消息发送至其他节点
* **TcpAcceptor**
  * 创建一个基于TCP协议的socket对象
  * 线程负责接收来自客户端的连接请求，并将连接信息放入队列，交由TcpRead线程处理
* **TcpRead**
  * 启动TcpAcceptor线程
  * 启动EventLoop，EventLoop采用epoll方式
  * EventLoop负责将TcpAcceptor中的请求转换为MessageEvent，并等待TcpClient发送消息
  * EventLoop负责接收来自不同TcpClient的消息，并通知Instance处理\(根据消息中的group id找到正确的group\)
* **TcpWrite**
  * 启动EventLoop
  * TcpClient负责接收所有Instance发送的TCP消息，预先放入消息队列，并通知EventLoop立即处理\(通过Pipe通信通知\)
  * TcpClient将消息放入到消息队列前，首先建立和服务端基于TCP协议的Scoket通信

  * 线程负责将消息队列中的消息发送至其他节点

图4

### 数据存储

PhxPaxos的数据存储抽象接口为LogStorage，内置实现类为MultiDatabase。MultiDatabase中针对每个Group创建了一个Databse对象负责真正的数据存储，它只是Database一个很薄的封装层。Database内部也并无过多的处理逻辑，其直接使用了高性能的本地数据库LevelDB，关于LevelDB的详细分析可参见我的系列文章《LevelDB源码剖析》。

数据存储模块的职责如下：

* Acceptor中所有accept数据存储，其中最主要的为instance id和value
* 存储其他PhxPaxos需要持久化的信息，如MasterVariables、SystemVariables等

### Paxos协议实现

## 质量属性

### 可靠性

### 性能




