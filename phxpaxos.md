# PhxPaxos

### 架构设计

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

### Node

一个Node对应一个物理节点，如果需要正确的运行Paxos协议，必须由以下部分组成：

1. **网络模块**：负责节点间网络通信。
2. **数据存储**：负责记录确定的有序值。
3. **Paxos分组**：上一节提到的一个Node同时运行多个Group的能力。
4. **选主服务**：用于确定Paxos的主节点。

> 主节点不是必须的，但选主后仅由主节点发起服务可以有效的提升性能。
>
> 选主服务有两点比较特殊：
>
> 1. 本身使用了Paxos协议
> 2. 每个Group需要单独选主\(存在另外一种实现：选主服务下放到Paxos分组中去完成\)





