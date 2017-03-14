# 、Paxos made simple释译

## 1 Introduction

The Paxos algorithm for implementing a fault-tolerant distributed system has been regarded as difficult to understand, perhaps because the original presentation was Greek to many readers \[5\].

In fact, it is among the simplest and most obvious of distributed algorithms. At its heart is a consensus algorithm—the “synod” algorithm of \[5\]. The next section shows that this consensus algorithm follows almost unavoidably from the properties we want it to satisfy.

The last section explains the complete Paxos algorithm, which is obtained by the straightforward application of consensus to the state machine approach for building a distributed system—an approach that should be well-known, since it is the subject of what is probably the most often-cited article on the theory of distributed systems \[4\].

## 2 The Consensus Algorithm

### 2.1 The Problem

Assume a collection of **processes** that can **propose** values. A consensus algorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen. If a value has been chosen, then processes should be able to learn the chosen value. The safety requirements for consensus are:

• Only a value that has been proposed may be chosen,

**只有被推举的值会被选中（chosen）**

• Only a single value is chosen, and

**只有一个值会被选中（chosen）**

• A **process** never **learns** that a value has been chosen unless it actually has been.

**直到一个值确实被选中（chosen），process才可以习得（learn）这个被选中的值**

> process对应一个进程，至少充当了learner、proposer两种角色：
>
> 1. propose是由proposer执行的动作
> 2. learn是由learner执行的动作

We won’t try to specify precise liveness requirements. However, the goal is to ensure that some proposed value is eventually chosen and, if a value has been chosen, then a process can eventually learn the value.

> 上面一直在讲一件事：
>
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

Since a proposal must be issued by a proposer before it can be accepted by an acceptor, P2b implies P2a, which in turn implies P2. To discover how to satisfy P2b, let’s consider how we would prove that it holds. We would assume that some proposal with number m and value v is chosen and show that any proposal issued with number n &gt; m also has value v. We would make the proof easier by using induction on n, so we can prove that proposal number n has value v under the additional assumption that every proposal issued with a number in m . . \(n − 1\) has value v, where i . . j denotes the set of numbers from i through j . For the proposal numbered m to be chosen, there must be some set C consisting of a majority of acceptors such that every acceptor in C accepted it. Combining this with the induction assumption, the hypothesis that m is chosen implies:

Every acceptor in C has accepted a proposal with number in的的  
 m . . \(n − 1\), and every proposal with number in m . . \(n − 1\)  
 accepted by any acceptor has value v.

Since any set S consisting of a majority of acceptors contains at least one  
 member of C , we can conclude that a proposal numbered n has value v by  
 ensuring that the following invariant is maintained:

**P2c. For any v and n, if a proposal with value v and number n is issued,  
 then there is a set S consisting of a majority of acceptors such that  
 either \(a\) no acceptor in S has accepted any proposal numbered less  
 than n, or \(b\) v is the value of the highest-numbered proposal among  
 all proposals numbered less than n accepted by the acceptors in S.    
**

P2c.对于任意的n和v，如果编号为n、value值为v的提案被提出，那么肯定存在一个由半数以上的acceptor组成的集合S，可以满足条件a\)或者b\)中的一个：a\) S中不存在任何的acceptor通过过编号小于n的提案. b\) v是S中所有acceptor通过的编号小于n的具有最大编号的提案的value值.

We can therefore satisfy P2b by maintaining the invariance of P2c.  
 To maintain the invariance of P2c, a proposer that wants to issue a pro  
posal numbered n must learn the highest-numbered proposal with number  
 less than n, if any, that has been or will be accepted by each acceptor in

some majority of acceptors. Learning about proposals already accepted is

easy enough; predicting future acceptances is hard. Instead of trying to pre

dict the future, the proposer controls it by extracting a promise that there

won’t be any such acceptances. In other words, the proposer requests that

the acceptors not accept any more proposals numbered less than n. This

leads to the following algorithm for issuing proposals.

1. A proposer chooses a new proposal number n and sends a request to

each member of some set of acceptors, asking it to respond with:

\(a\) A promise never again to accept a proposal numbered less than

n, and

\(b\) The proposal with the highest number less than n that it has

accepted, if any.

I will call such a request a prepare request with number n.

1. If the proposer receives the requested responses from a majority of

the acceptors, then it can issue a proposal with number n and value

v, where v is the value of the highest-numbered proposal among the

responses, or is any value selected by the proposer if the responders

reported no proposals.

