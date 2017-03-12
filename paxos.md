# Paxos made simple释译

## 1 Introduction

The Paxos algorithm for implementing a fault-tolerant distributed system has been regarded as difficult to understand, perhaps because the original presentation was Greek to many readers \[5\]. 

In fact, it is among the simplest and most obvious of distributed algorithms. At its heart is a consensus algorithm—the “synod” algorithm of \[5\]. The next section shows that this consensus algorithm follows almost unavoidably from the properties we want it to satisfy. 

The last section explains the complete Paxos algorithm, which is obtained by the straightforward application of consensus to the state machine approach for building a distributed system—an approach that should be well-known, since it is the subject of what is probably the most often-cited article on the theory of distributed systems \[4\].

## 2 The Consensus Algorithm

### 2.1 The Problem

Assume a collection of processes that can propose values. A consensus al

gorithm ensures that a single one among the proposed values is chosen. If

no value is proposed, then no value should be chosen. If a value has been

chosen, then processes should be able to learn the chosen value. The safety

requirements for consensus are:

• Only a value that has been proposed may be chosen,

• Only a single value is chosen, and

• A process never learns that a value has been chosen unless it actually

has been.

We won’t try to specify precise liveness requirements. However, the goal is

to ensure that some proposed value is eventually chosen and, if a value has

been chosen, then a process can eventually learn the value.

We let the three roles in the consensus algorithm be performed by three

classes of agents: proposers, acceptors, and learners. In an implementation,

a single process may act as more than one agent, but the mapping from

agents to processes does not concern us here.

Assume that agents can communicate with one another by sending mes

sages. We use the customary asynchronous, non-Byzantine model, in which:

