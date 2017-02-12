# PhxPaxos

### 架构设计

Paxos为分布式一致性协议，用于解决异步通信环境下分布式系统的一致性问题。做为分布式系统，必定存在多个节点共同参与\(一般来说节点数量为奇数个，常用的节点数量如3，5等\)。

一个Paxos实例执行可以确定一个值，其最实际的工程价值莫过于提供选主服务。但如果可以有序的确定多个值，再结合状态机将值赋予业务含义\(如日志回放等\)，Paxos将具有更大的价值。

PhxPaxos基于《Paxos made simple》实现，可以做到有序确定多个值，更进一步的是其可以同时确定多组有序的值。

PhxPaxos整体架构如下：

![](/assets/整体架构.png)

* 一个Group用于有序的确定多个值，多个Group可以同时确定多组有序的值。
* Group中的每个Node代表一个物理节点\(VM或进程\)，一个Node中同时运行了多个Group的Instance，如下表

|  | Group 0 | Group 1 | Group 2 |
| :--- | :--- | :--- | :--- |
| Node A | Instance X | Instance Y | Instance Z |
| Node B | Instance X | Instance Y | Instance Z |
| Node C | Instance X | Instance Y | Instance Z |

* 每个Instance为Paxos协议的执行主体，用于确定一个值，当Instance N的值被确定后，Instance N被销毁，构建Instance N+1确定下一个值。

### Node

的多得多得多得多的





























