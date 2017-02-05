
---

# Paxos made simple释译

## 1 Introduction

The Paxos algorithm for implementing a fault-tolerant distributed system has been regarded as difficult to understand, perhaps because the original presentation was Greek to many readers \[5\].

In fact, it is among the simplest and most obvious of distributed algorithms. At its heart is a consensus algorithm—the “synod” algorithm of \[5\]. The next section shows that this consensus algorithm follows almost unavoidably from the properties we want it to satisfy.

The last section explains the complete Paxos algorithm, which is obtained by the straightforward application of consensus to the state machine approach for building a distributed system—an approach that should be well-known, since it is the subject of what is probably the most often-cited article on the theory of distributed systems \[4\].

## 2 The Consensus Algorithm

### 2.1 The Problem

Assume a collection of **processes** that can **propose** values. A consensus algorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen. If a value has been chosen, then processes should be able to learn the chosen value. The safety requirements for consensus are:

• Only a value that has been proposed may be chosen,

**只有提案值的值会被选中（chosen）**

• Only a single value is chosen, and

**只有一个值会被选中（chosen）**

• A **process** never **learns** that a value has been chosen unless it actually has been.

**直到一个值确实被选中（chosen），process才可以习得（learn）这个被选中的值**

> process对应一个进程，至少充当了learner、proposer两种角色：
>
> 1. propose是由proposer执行的动作
> 2. learn是由learner执行的动作
>
> 后面会看到，其实每个进程被设计成一个完整的paxos协议执行体，包括proposer、acceptor、learner等.

We won’t try to specify precise liveness requirements. However, the goal is to ensure that some proposed value is eventually chosen and, if a value has been chosen, then a process can eventually learn the value.

> 当多个process发起多个提议时，paxos确保只有一个提议值（value）被选中（chosen），并且其他process可以习得（learn）这个选中值，从而确保多个process在最终达成一致

We let the three roles in the consensus algorithm be performed by three classes of agents: **proposers**, **acceptors**, and **learners**. In an implementation, a single process may act as more than one agent, but the mapping from agents to processes does not concern us here. Assume that agents can communicate with one another by sending messages. We use the customary asynchronous, non-Byzantine model, in which:

• Agents operate at arbitrary speed, may fail by stopping, and may  
 restart. Since all agents may fail after a value is chosen and then  
 restart, a solution is impossible unless some information can be re  
membered by an agent that has failed and restarted.

• Messages can take arbitrarily long to be delivered, can be duplicated,  
 and can be lost, but they are not corrupted.

> 将paxos按职责划分为多个角色：proposer、acceptor、learner。
>
> 角色间通过消息通信（非拜占庭式消息--不会被篡改），消息可丢失、重复、延迟。

### 2.2 Choosing a Value

The easiest way to choose a value is to have a single acceptor agent. A proposer sends a proposal to the acceptor, who chooses the first proposed value that it receives. Although simple, this solution is unsatisfactory because the failure of the acceptor makes any further progress impossible. So, let’s try another way of choosing a value. Instead of a single acceptor, let’s use multiple acceptor agents. A proposer sends a proposed value to a set of acceptors. An acceptor may accept the proposed value. The value is chosen when a large enough set of acceptors have accepted it. How large is large enough? To ensure that only a single value is chosen, we can let a large enough set consist of any majority of the agents. Because any two majorities  
 have at least one acceptor in common, this works if an acceptor can accept at most one value. \(There is an obvious generalization of a majority that has been observed in numerous papers, apparently starting with \[3\].\). In the absence of failure or message loss, we want a value to be chosen even if only one value is proposed by a single proposer. This suggests the  requirement:

**P1. An acceptor must accept the first proposal that it receives.**

**P1：一个acceptor必须通过\(accept\)它收到的第一个提案**

But this requirement raises a problem. Several values could be proposed by different proposers at about the same time, leading to a situation in which every acceptor has accepted a value, but no single value is accepted by a majority of them. Even with just two proposed values, if each is accepted by about half the acceptors, failure of a single acceptor could make it impossible  
 to learn which of the values was chosen.

P1 and the requirement that a value is chosen only when it is accepted by a majority of acceptors imply that **an acceptor must be allowed to accept more than one proposal**. We keep track of the different proposals that an acceptor may accept by assigning a \(natural\) number to each proposal, so a proposal consists of a proposal number and a value. To prevent confusion, we require that different proposals have different numbers. How this is achieved depends on the implementation, so for now we just assume it. A value is chosen when a single proposal with that value has been accepted by a majority of the acceptors. In that case, we say that the proposal \(as well as its value\) has been chosen. We can allow multiple proposals to be chosen, but we must guarantee that all chosen proposals have the same value. By induction on the proposal number, it suffices to guarantee:

**P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.**

**P2.如果具有value值v的提案被选定\(chosen\)了，那么所有比它编号更高的被选定的提案的value值也必须是v**

> 注意：这里强调的chosen

Since numbers are totally ordered, condition P2 guarantees the crucial safety property that only a single value is chosen. To be chosen, a proposal must be accepted by at least one acceptor. So, we can satisfy P2 by satisfying:

**P2a. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.**

**P2a. 如果具有value值v的提案被选定\(chosen\)了，那么所有比它编号更高的被acceptor通过的提案的value值也必须是v**

> 注意：这里强调的是accpet

We still maintain P1 to ensure that some proposal is chosen. Because communication is asynchronous, a proposal could be chosen with some particular acceptor c never having received any proposal. Suppose a new proposer “wakes up” and issues a higher-numbered proposal with a different value. P1 requires c to accept this proposal, violating P2a. Maintaining both P1

and P2a requires strengthening P2a to:

**P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v.**

**P2b.如果具有value值v的提案被选定，那么所有比它编号更高的被proposer提出的提案的value值也必须是v**

> 注意：这里强调的是issued
>
> chosen-&gt;accept-&gt;issued

Since a proposal must be issued by a proposer before it can be accepted by an acceptor, P2b implies P2a, which in turn implies P2. To discover how to satisfy P2b, let’s consider how we would prove that it holds. We would assume that some proposal with number m and value v is chosen and show that any proposal issued with number n &gt; m also has value v. We would make the proof easier by using induction on n, so we can prove that proposal number n has value v under the additional assumption that every proposal issued with a number in m . . \(n − 1\) has value v, where i . . j denotes the set of numbers from i through j . For the proposal numbered m to be chosen, there must be some set C consisting of a majority of acceptors such that every acceptor in C accepted it. Combining this with the induction assumption, the hypothesis that m is chosen implies:

Every acceptor in C has accepted a proposal with number in m . . \(n − 1\), and every proposal with number in m . . \(n − 1\)  
 accepted by any acceptor has value v.

Since any set S consisting of a majority of acceptors contains at least one member of C , we can conclude that a proposal numbered n has value v by ensuring that the following invariant is maintained:

**P2c. For any v and n, if a proposal with value v and number n is issued, then there is a set S consisting of a majority of acceptors such that either \(a\) no acceptor in S has accepted any proposal numbered less than n, or \(b\) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S.  
P2c.对于任意的n和v，如果编号为n、value值为v的提案被提出，那么肯定存在一个由半数以上的acceptor组成的集合S，可以满足条件a\)或者b\)中的一个：a\) S中不存在任何的acceptor通过过编号小于n的提案. b\) v是S中所有acceptor通过的编号小于n的具有最大编号的提案的value值.**

We can therefore satisfy P2b by maintaining the **invariance **of P2c. To maintain the **invariance** of P2c, a proposer that wants to issue a proposal numbered n must learn the highest-numbered proposal with number less than n, if any, that has been or will be accepted by each acceptor in some majority of acceptors. Learning about proposals already accepted is easy enough; predicting future acceptances is hard. Instead of trying to predict the future, the proposer controls it by extracting a promise that there won’t be any such acceptances. In other words, the proposer requests that the acceptors not accept any more proposals numbered less than n. This leads to the following algorithm for issuing proposals.

1. **A proposer chooses a new proposal number n and sends a request to each member of some set of acceptors, asking it to respond with:**  
   1. **A promise never again to accept a proposal numbered less than n, and **  
   2. **The proposal **_**value**_** with the highest number less than n that it has accepted, if any. **

   **I will call such a request a prepare request with number n.**

2. **If the proposer receives the requested responses from a majority of the acceptors, then it can issue a proposal with number n and value v, where v is the value of the highest-numbered proposal among the responses, or is any value selected by the proposer if the responders reported no proposals.**

A proposer issues a proposal by sending, to some set of acceptors, a request that the proposal be accepted. \(This need not be the same set of acceptors that responded to the initial requests.\) Let’s call this an accept request.

> 一个完整的提案由{编号n，提案值v}两部分组成，确定提案的流程如下：
>
> 1. proposer选择新的编号n，并将其发送到所有的acceptor
> 2. acceptor接收到请求后完成如下工作：
>    1. 承诺\(promise\)：承诺不再accept所有编号小于n的请求，即reject编号小于n的请求
>    2. 返回\(reply\)：返回当前小于n的最大编号所对应的提议值v\(如果存在\)
> 3. 如果proposer收到了超过半数的acceptor响应，此时才可以真正的发起提案，否则本轮提案以失败结束
> 4. 提案的编号为n，提案值为v，如果提议值v不存在，则可由proposer指定任意值
> 5. 至此，提案{编号n，提案值v}确定
>
> 这里，有一个关键的信息是：保持P2c的不变性

This describes a proposer’s algorithm. What about an acceptor? It can receive two kinds of requests from proposers: prepare requests and accept requests. An acceptor can ignore any request without compromising safety. So, we need to say only when it is allowed to respond to a request. It can always respond to a prepare request. It can respond to an accept request, accepting the proposal, iff it has not promised not to. In other words:

**P1a. An acceptor can accept a proposal numbered n iff it has not responded to a prepare request having a number greater than n.  
P1a.一个acceptor可以接受一个编号为n的提案，只要它还未响应任何编号大于n的prepare请求**

Observe that P1a subsumes P1. We now have a complete algorithm for choosing a value that satisfies the required safety properties—assuming unique proposal numbers. The final algorithm is obtained by making one small optimization. Suppose an acceptor receives a prepare request numbered n, but it has already responded to a prepare request numbered greater than n, thereby promising not to accept any new proposal numbered n. There is then no reason for the acceptor to respond to the new prepare request, since it will not accept the proposal numbered n that the proposer wants to issue. So we have the acceptor ignore such a prepare request.** We also have it ignore a prepare request for a proposal it has already accepted.**

> **P1：一个acceptor必须通过\(accept\)它收到的第一个提案**
>
> **P1a.一个acceptor可以接受一个编号为n的提案，只要它还未响应任何编号大于n的prepare请求**
>
> 这里针对P1的定义做了进一步的强化

With this optimization, an acceptor needs to **remember** only the **highestnumbered** proposal that it has ever accepted and the number of the highestnumbered prepare request to which it has responded. **Because P2c must be kept invariant regardless of failures, an acceptor must remember this information even if it fails and then restarts**. Note that the proposer can always abandon a proposal and forget all about it—as long as it never tries to issue another proposal with the same number.

> P2C的不变性在acceptor异常、重启等状态下得以保存，因此acceptor必须将该信息持久化

Putting the actions of the proposer and acceptor together, we see that the algorithm operates in the following two phases.

**Phase 1. **

\(a\) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors.

\(b\) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal _**value**_\(if any\) that it has accepted.

**Phase 2. **

\(a\) If the proposer receives a response to its prepare requests\(numbered n\) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals.

\(b\) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.

A proposer can make multiple proposals, so long as it follows the algorithm for each one. It can abandon a proposal in the middle of the protocol at any time. \(Correctness is maintained, even though requests and/or responses for the proposal may arrive at their destinations long after the proposal was abandoned.\) It is probably a good idea to abandon a proposal if some  
 proposer has begun trying to issue a higher-numbered one. Therefore, if an acceptor ignores a prepare or accept request because it has already received a prepare request with a higher number, then it should probably inform the proposer, who should then abandon its proposal. This is a performance optimization that does not affect correctness.

> 注意：本节讲述的是如何chosen一个value
>
> 之所以强调value，是因为proposal由number、value两部分组成，但我们只关心value，而不关心number，也不关心proposal
>
> 也就是说，我们可能提了多个不同的提案\(由不同的编号\)，最终chosen的提案也可能有多个，但chosen的value只有一个

### 2.3 Learning a Chosen Value

To learn that a value has been chosen, a learner must find out that a proposal has been accepted by a majority of acceptors. The obvious algorithm is to have each acceptor, whenever it accepts a proposal, respond to all learners, sending them the proposal. This allows learners to find out about a chosen value as soon as possible, but it requires each acceptor to respond to each learner—a number of responses equal to the product of the number of acceptors and the number of learners.

The assumption of non-Byzantine failures makes it easy for one learner  to find out from another learner that a value has been accepted. We can have the acceptors respond with their acceptances to a distinguished learner, which in turn informs the other learners when a value has been chosen. This approach requires an extra round for all the learners to discover the chosen  
 value. It is also less reliable, since the distinguished learner could fail. But it requires a number of responses equal only to the sum of the number of acceptors and the number of learners.

More generally, the acceptors could respond with their acceptances to some set of distinguished learners, each of which can then inform all the learners when a value has been chosen. Using a larger set of distinguished learners provides greater reliability at the cost of greater communication complexity.

Because of message loss, a value could be chosen with no learner ever finding out. The learner could ask the acceptors what proposals they have accepted, but failure of an acceptor could make it impossible to know whether or not a majority had accepted a particular proposal. In that case, learners will find out what value is chosen only when a new proposal is chosen. If  
 a learner needs to know whether a value has been chosen, it can have a proposer issue a proposal, using the algorithm described above.

> 该协议要求：P2C中acceptor每次accept一个新的提案{n,v}，则该提案需要被落盘。由此保证P2C得以正确进行下去
>
> 假设有n个节点参与了paxos协议，在网络通信正常、节点工作正常情况下。
>
> 在Learn选定值时，作者的想法是：
>
> a\) 可以在每次accept一个提案时，通知所有的learner：
>
> * 每个acceptor在accept一个提案时广播到所有learner，由此带来的网络消息总数为n\*n
> * accept的提案不一定是chosen的提案，如果经历了m次accept才最终hosen一个提案，那么网络消息总数为m\*n\*n
> * chosen value确认：超过半数的acceptor向learner发来了相同的accepted value，该accepted value即为chosen value
>
>   b\) 每次accpet一个提案时，通知到一个主leanrer，
>
> * chosen value确认：超过半数的acceptor向主learner发来了相同的accepted value，该accepted value即为chosen value
>
> * 主learner异常导致其他Learner学不到chosen值
>
> c\) 每次accept一个提案时，通知到K个主learner  
> d\) learner主动向所有的acceptor发起learn请求，习得chosen值，但由于acceptor中持久化的值仅仅保证是accepted值，并不一定是chosen值，因此，这里需要由learner充当proposal发起一次提案，用于习得chosen值
>
> 上述4中方案均满足paxos开篇三原则中的第三条：
>
> • A **process** never **learns** that a value has been chosen unless it actually has been.
>
> **直到一个值确实被选中（chosen），process才可以习得（learn）这个被选中的值**
>
> 按我的理解，作者讲述的learner习得过程虽然可以达到learner的效果，但是并不是非常好
>
> * learner应该直接接收chosen value，而不是每次接收所有的accepted value，并自行再做一次chosen value确定。
> * 一个值是否被chosen，最先是被proposer获知的，因此由proposer通知learner更合理

### 2.4 Progress

It’s easy to construct a scenario in which two proposers each keep issuing a sequence of proposals with increasing numbers, none of which are ever chosen. Proposer p completes phase 1 for a proposal number n1. Another proposer q then completes phase 1 for a proposal number n2 &gt; n1. Proposer p’s phase 2 accept requests for a proposal numbered n1 are ignored because the acceptors have all promised not to accept any new proposal numbered less than n2. So, proposer p then begins and completes phase 1 for a new proposal number n3 &gt; n2, causing the second phase 2 accept requests of proposer q to be ignored. And so on.

To guarantee progress, a distinguished proposer must be selected as the only one to try issuing proposals. If the distinguished proposer can communicate successfully with a majority of acceptors, and if it uses a proposal with number greater than any already used, then it will succeed in issuing aproposal that is accepted. By abandoning a proposal and trying again if it learns about some request with a higher proposal number, the distinguished proposer will eventually choose a high enough proposal number.

If enough of the system \(proposer, acceptors, and communication network\) is working properly, liveness can therefore be achieved by electing a single distinguished proposer. The famous result of Fischer, Lynch, and Patterson \[1\] implies that a reliable algorithm for electing a proposer must use either randomness or real time—for example, by using timeouts. However,  
 safety is ensured regardless of the success or failure of the election.

> 这里讲到一种可能导致paxos陷入死循环的场景，并提出了一种简单的解决方案：选择一个主proposer，所有的提议由主proposer发起

### 2.5 The Implementation

The Paxos algorithm \[5\] assumes a network of processes. In its consensus algorithm, each process plays the role of proposer, acceptor, and learner. The algorithm chooses a leader, which plays the roles of the distinguished proposer and the distinguished learner.

> 创建一组通过网络通信的进程，每个进程同时扮演了proposer、acceptor、learner，在这些进程中选择一个主进程，做为主proposer和主learner。

The Paxos consensus algorithm is precisely the one described above, where requests and responses are sent as ordinary messages. \(Response messages are tagged with the corresponding proposal number to prevent confusion.\) Stable storage, preserved during failures, is used to maintain the information that the acceptor must remember. **An acceptor records its intended response in stable storage beforeactually sending the response.**

> acceptor在返回响应前必须做数据持久化，以免数据丢失。

All that remains is to describe the mechanism for guaranteeing that no two proposals are ever issued with the same number. Different proposers choose their numbers from disjoint sets of numbers, so two different proposers never issue a proposal with the same number. Each proposer remembers \(in stable storage\) the highest-numbered proposal it has tried to issue, and begins phase 1 with a higher proposal number than any it has already used.

> 通过算法保证每台机器之间编号唯一且严格递增

### 3 Implementing a State Machine

A simple way to implement a distributed system is as a collection of clients that issue commands to a central server. The server can be described as a deterministic state machine that performs client commands in some sequence. The state machine has a current state; it performs a step by taking as input a command and producing an output and a new state. For example, the clients of a distributed banking system might be tellers, and the state-machine state might consist of the account balances of all users.

A withdrawal would be performed by executing a state machine command that decreases an account’s balance if and only if the balance is greater than the amount withdrawn, producing as output the old and new balances.An implementation that uses a single central server fails if that server fails. We therefore instead use a collection of servers, each one independently implementing the state machine. Because the state machine is deterministic, all the servers will produce the same sequences of states and outputs if they all execute the same sequence of commands. A client issuing a command can then use the output generated for it by any server.

To guarantee that all servers execute the same sequence of state machine commands, we implement a sequence of separate instances of the Paxos consensus algorithm, the value chosen by the i th instance being the i th state machine command in the sequence. Each server plays all the roles \(proposer, acceptor, and learner\) in each instance of the algorithm. For now, I assume  
 that the set of servers is fixed, so all instances of the consensus algorithm use the same sets of agents.In normal operation, a single server is elected to be the leader, which acts as the distinguished proposer \(the only one that tries to issue proposals\) in all instances of the consensus algorithm. Clients send commands to the leader, who decides where in the sequence each command should appear. If the leader decides that a certain client command should be the 135th command, it tries to have that command chosen as the value of the 135th instance of the consensus algorithm. It will usually succeed. It might fail because of failures, or because another server also believes itself to be the leader and has a different idea of what the 135th command should be. But the consensus algorithm ensures that at most one command can be chosen as the 135th one.

Key to the efficiency of this approach is that, in the Paxos consensus algorithm, the value to be proposed is not chosen until phase 2. Recall that, after completing phase 1 of the proposer’s algorithm, either the value to be proposed is determined or else the proposer is free to propose any value.

> Paxos协议也建议选择一个leader节点，所有的提案都有leader发起，以此减少可能出现的提案冲突，进而提升性能。
>
> Paxos也采用了“主备”模式，那么它和典型的主备模式有何区别呢？
>
> **答**：Paxos中选举的主只是为了提升性能，在出现双主、甚至多主时依旧可以正常工作，并保证分布式系统的一致性

I will now describe how the Paxos state machine implementation works during normal operation. Later, I will discuss what can go wrong. I consider what happens when the previous leader has just failed and a new leader has been selected. \(System startup is a special case in which no commands have yet been proposed.\)

The new leader, being a learner in all instances of the consensus algorithm, should know most of the commands that have already been chosen. Suppose it knows commands 1–134, 138, and 139—that is, the values chosen in instances 1–134, 138, and 139 of the consensus algorithm. \(We will see later how such a gap in the command sequence could arise.\) It then  
 executes phase 1 of instances 135–137 and of all instances greater than 139.\(I describe below how this is done.\) Suppose that the outcome of these executions determine the value to be proposed in instances 135 and 140, but leaves the proposed value unconstrained in all other instances. The leader then executes phase 2 for instances 135 and 140, thereby choosing commands  
 135 and 140.

> 135-137，139以上执行阶段1，其中135，140正确返回，进而执行阶段2确定135，140两个值。
>
> 注意：这里指的是139以上的全部commands，这是怎么做到的呢？请往后看

The leader, as well as any other server that learns all the commands the leader knows, can now execute commands 1–135. However, it can’t execute commands 138–140, which it also knows, because commands 136 and 137 have yet to be chosen. The leader could take the next two commands requested by clients to be commands 136 and 137. Instead, we let it fill the gap immediately by proposing, as commands 136 and 137, a special “noop” command that leaves the state unchanged. \(It does this by executing phase 2 of instances 136 and 137 of the consensus algorithm.\) Once these no-op commands have been chosen, commands 138–140 can be executed.

> 至此，140之前的value只有136和137未知，因此状态机无法执行至140，必须确定136，137的值。
>
> 这里采用了作者称之为noop方法，即只执行阶段2，将随之而来的两个客户端请求做为136，137的选定值。

Commands 1–140 have now been chosen. The leader has also completed phase 1 for all instances greater than 140 of the consensus algorithm, and it is free to propose any value in phase 2 of those instances. It assigns command number 141 to the next command requested by a client, proposing it as the value in phase 2 of instance 141 of the consensus algorithm. It proposes the next client command it receives as command 142, and so on.

The leader can propose command 142 before it learns that its proposed command 141 has been chosen. It’s possible for all the messages it sent in proposing command 141 to be lost, and for command 142 to be chosen before any other server has learned what the leader proposed as command 141. When the leader fails to receive the expected response to its phase 2 messages in instance 141, it will retransmit those messages. If all goes well, its proposed command will be chosen. However, it could fail first, leaving a gap in the sequence of chosen commands. In general, suppose a leader can get α commands ahead—that is, it can propose commands i + 1 through i +α after commands 1 through i are chosen. A gap of up to α−1 commands could then arise.

> leader同时处理r个commond，应该是出于性能考虑在实现上做的一种优化

A newly chosen leader executes phase 1 for infinitely many instances of the consensus algorithm—in the scenario above, for instances 135–137 and all instances greater than 139. Using the same proposal number for all instances, it can do this by sending a single reasonably short message to the other servers. In phase 1, an acceptor responds with more than a simple OK only if it has already received a phase 2 message from some proposer. \(In the scenario, this was the case only for instances 135 and 140.\) Thus, a server \(acting as acceptor\) can respond for all instances with a single reasonably short message. Executing these infinitely many instancesof phase 1 therefore poses no problem.

Since failure of the leader and election of a new one should be rare events, the effective cost of executing a state machine command—that is, of achieving consensus on the command/value—is the cost of executing only phase 2 of the consensus algorithm. It can be shown that phase 2 of the Paxos consensus algorithm has the minimum possible cost of any algorithm

for reaching agreement in the presence of faults \[2\]. Hence, the Paxos algorithm is essentially optimal.

> 在前两章讲述的流程中，基于无主模式确定一个值的方法为：  
> 1. 执行1阶段的prepare动作，尝试选定一个提案  
> 2. 为该提案指定一个值
>
> 当前，我们已经为集群选定了一个主节点，并且只能由主节点发起一个提案。  
> 此时，提案不会再冲突，主节点选定的提案必然为最终的提案\(当然也不完全是这样，不然就不会存在gap\)。  
> 在单主模式下，确定一个值的方法为：
>
> 1. 只执行阶段2，确定一个值
>
> 在原主异常，选定新主时，为所有未确定的instance，采用一个特殊的命令执行一次1阶段，1阶段执行失败的instance（如135，140）将使用已有值做为chosen value。1阶段执行成功的instance采用单主模式执行阶段2

This discussion of the normal operation of the system assumes that there is always a single leader, except for a brief period between the failure of the current leader and the election of a new one. In abnormal circumstances, the leader election might fail. If no server is acting as leader, then no new commands will be proposed. If multiple servers think they are leaders, then they can all propose values in the same instance of the consensus algorithm, which could prevent any value from being chosen. However, safety is preserved—two different servers will never disagree on the value chosen as the i th state machine command. Election of a single leader is needed only to ensure progress.

If the set of servers can change, then there must be some way of determining what servers implement what instances of the consensus algorithm. The easiest way to do this is through the state machine itself. The current set of servers can be made part of the state and can be changed with ordinary state-machine commands. We can allow a leader to get α commands ahead by letting the set of servers that execute instance i + α of the consensus algorithm be specified by the state after execution of the i th state machine command. This permits a simple implementation of an arbitrarily sophisticated reconfiguration algorithm.



