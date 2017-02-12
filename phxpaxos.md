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

图390

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

Paxos协议中规定了三类角色：Proposer、Accetor、Learner。协议实现完全基于《Paxos made simple》，实现上也并不复杂，来看代码实现：

**Proposer**

```cpp
    //发起提案的入口函数
    int Proposer :: NewValue ( const std::string & sValue )
    {
        BP->GetProposerBP()->NewProposal ( sValue );

        //记录本次的提案值
        if ( m_oProposerState.GetValue().size() == 0 )
        {
            m_oProposerState.SetValue ( sValue );
        }

        m_iLastPrepareTimeoutMs = START_PREPARE_TIMEOUTMS;
        m_iLastAcceptTimeoutMs = START_ACCEPT_TIMEOUTMS;

        //允许跳过Prepare阶段，直接进入Accept
        if ( m_bCanSkipPrepare && !m_bWasRejectBySomeone )
        {
            BP->GetProposerBP()->NewProposalSkipPrepare();

            PLGHead ( "skip prepare, directly start accept" );
            Accept();
        }
        else    //从Prepare阶段开始Paxos协议
        {
            //if not reject by someone, no need to increase ballot
            Prepare ( m_bWasRejectBySomeone );
        }

        return 0;
    }

    //Prepare阶段
    void Proposer :: Prepare ( const bool bNeedNewBallot )
    {
        PLGHead ( "START Now.InstanceID %lu MyNodeID %lu State.ProposalID %lu State.ValueLen %zu",
                  GetInstanceID(), m_poConfig->GetMyNodeID(), m_oProposerState.GetProposalID(),
                  m_oProposerState.GetValue().size() );

        BP->GetProposerBP()->Prepare();
        m_oTimeStat.Point();

        ExitAccept();
        m_bIsPreparing = true;
        m_bCanSkipPrepare = false;
        m_bWasRejectBySomeone = false;

        m_oProposerState.ResetHighestOtherPreAcceptBallot();

        //分配一个新的提案编号：Porposal ID
        if ( bNeedNewBallot )
        {
            m_oProposerState.NewPrepare();
        }

        //发起提案：instance id、node id、proposal id
        PaxosMsg oPaxosMsg;
        oPaxosMsg.set_msgtype ( MsgType_PaxosPrepare );
        oPaxosMsg.set_instanceid ( GetInstanceID() );
        oPaxosMsg.set_nodeid ( m_poConfig->GetMyNodeID() );
        oPaxosMsg.set_proposalid ( m_oProposerState.GetProposalID() );

        m_oMsgCounter.StartNewRound();

        //添加定时器，当出现超时后，重新触发Prepare
        AddPrepareTimer();

        PLGHead ( "END OK" );

        BroadcastMessage ( oPaxosMsg );
    }

    //来自各个Acceptor的响应消息
    void Proposer :: OnPrepareReply ( const PaxosMsg & oPaxosMsg )
    {
        PLGHead ( "START Msg.ProposalID %lu State.ProposalID %lu Msg.from_nodeid %lu RejectByPromiseID %lu",
                  oPaxosMsg.proposalid(), m_oProposerState.GetProposalID(),
                  oPaxosMsg.nodeid(), oPaxosMsg.rejectbypromiseid() );

        BP->GetProposerBP()->OnPrepareReply();

        //当前以不在Prepare阶段，不处理
        //1. 某个节点响应过慢，提案已进入到Accept阶段
        //2. 整个提案响应过慢，提案已终止
        if ( !m_bIsPreparing )
        {
            BP->GetProposerBP()->OnPrepareReplyButNotPreparing();
            //PLGErr("Not preparing, skip this msg");
            return;
        }

        if ( oPaxosMsg.proposalid() != m_oProposerState.GetProposalID() )
        {
            BP->GetProposerBP()->OnPrepareReplyNotSameProposalIDMsg();
            //PLGErr("ProposalID not same, skip this msg");
            return;
        }

        //记录已收到来自node id节点的响应
        m_oMsgCounter.AddReceive ( oPaxosMsg.nodeid() );

        //提案被接受
        if ( oPaxosMsg.rejectbypromiseid() == 0 )
        {
            BallotNumber oBallot ( oPaxosMsg.preacceptid(), oPaxosMsg.preacceptnodeid() );
            PLGDebug ( "[Promise] PreAcceptedID %lu PreAcceptedNodeID %lu ValueSize %zu",
                       oPaxosMsg.preacceptid(), oPaxosMsg.preacceptnodeid(), oPaxosMsg.value().size() );
            //记录接受提案的node id
            m_oMsgCounter.AddPromiseOrAccept ( oPaxosMsg.nodeid() );
            //记录接受提案节点返回的提案值
            m_oProposerState.AddPreAcceptValue ( oBallot, oPaxosMsg.value() );
        }
        else    //提案被拒绝
        {
            PLGDebug ( "[Reject] RejectByPromiseID %lu", oPaxosMsg.rejectbypromiseid() );
            //记录拒绝提案的node id
            m_oMsgCounter.AddReject ( oPaxosMsg.nodeid() );
            m_bWasRejectBySomeone = true;
            //记录拒绝提案节点返回的Proposal ID，以便下次基于此信息重新发起提案
            m_oProposerState.SetOtherProposalID ( oPaxosMsg.rejectbypromiseid() );
        }

        //超过半数接受该提案，提案通过
        if ( m_oMsgCounter.IsPassedOnThisRound() )
        {
            int iUseTimeMs = m_oTimeStat.Point();
            BP->GetProposerBP()->PreparePass ( iUseTimeMs );
            PLGImp ( "[Pass] start accept, usetime %dms", iUseTimeMs );
            //注意：这里做了一个优化，一旦Prepare阶段的提案被通过后，就自动跳过Prepare阶段，以减少网络传输落盘次数
            m_bCanSkipPrepare = true;
            Accept();
        }
        //超过半数拒绝该提案，或提案已全部响应，但并未超过半数通过(如4个节点，两个接受、两个拒绝)
        else if ( m_oMsgCounter.IsRejectedOnThisRound()
                  || m_oMsgCounter.IsAllReceiveOnThisRound() )
        {
            BP->GetProposerBP()->PrepareNotPass();
            PLGImp ( "[Not Pass] wait 30ms and restart prepare" );
            //设置定时器，并在10-40ms之后重新发起提案
            AddPrepareTimer ( OtherUtils::FastRand() % 30 + 10 );
        }

        PLGHead ( "END" );
    }
```

```cpp
    //Accept阶段
    void Proposer :: Accept()
    {
        PLGHead ( "START ProposalID %lu ValueSize %zu ValueLen %zu",
                  m_oProposerState.GetProposalID(), m_oProposerState.GetValue().size(), 
                  m_oProposerState.GetValue().size() );

        BP->GetProposerBP()->Accept();
        m_oTimeStat.Point();

        //并发控制，标记当前已进入Accept阶段 -- 无锁化
        ExitPrepare();
        m_bIsAccepting = true;

        PaxosMsg oPaxosMsg;
        oPaxosMsg.set_msgtype ( MsgType_PaxosAccept );
        oPaxosMsg.set_instanceid ( GetInstanceID() );
        oPaxosMsg.set_nodeid ( m_poConfig->GetMyNodeID() );
        oPaxosMsg.set_proposalid ( m_oProposerState.GetProposalID() );
        oPaxosMsg.set_value ( m_oProposerState.GetValue() );
        oPaxosMsg.set_lastchecksum ( GetLastChecksum() );

        m_oMsgCounter.StartNewRound();

        //添加定时器、用于Accept超时后重新进入Prepare阶段
        AddAcceptTimer();

        PLGHead ( "END" );

        //发送消息到其他节点
        BroadcastMessage ( oPaxosMsg, BroadcastMessage_Type_RunSelf_Final );
    }

    //Accept阶段响应
    void Proposer :: OnAcceptReply ( const PaxosMsg & oPaxosMsg )
    {
        PLGHead ( "START Msg.ProposalID %lu State.ProposalID %lu Msg.from_nodeid %lu RejectByPromiseID %lu",
                  oPaxosMsg.proposalid(), m_oProposerState.GetProposalID(),
                  oPaxosMsg.nodeid(), oPaxosMsg.rejectbypromiseid() );

        BP->GetProposerBP()->OnAcceptReply();

        //当前已不在Accept阶段
        //1. 某个节点响应过慢，提案已完成
        //2. 多个节点响应过慢，提案已重新进入Prepare阶段阶段
        //3. 整个提案响应过慢，提案已终止
        if ( !m_bIsAccepting )
        {
            //PLGErr("Not proposing, skip this msg");
            BP->GetProposerBP()->OnAcceptReplyButNotAccepting();
            return;
        }

        //提案编号不一致，跳过不处理
        //    proposal id不一致表明一定不是同一个instance id
        if ( oPaxosMsg.proposalid() != m_oProposerState.GetProposalID() )
        {
            //PLGErr("ProposalID not same, skip this msg");
            BP->GetProposerBP()->OnAcceptReplyNotSameProposalIDMsg();
            return;
        }

        //记录已收到node id节点的消息
        m_oMsgCounter.AddReceive ( oPaxosMsg.nodeid() );

        //提案被接收
        if ( oPaxosMsg.rejectbypromiseid() == 0 )
        {
            PLGDebug ( "[Accept]" );
            //记录接收提案的节点编号
            m_oMsgCounter.AddPromiseOrAccept ( oPaxosMsg.nodeid() );
        }
        else    //提案被拒绝
        {
            PLGDebug ( "[Reject]" );

            //记录拒绝提案的节点编号
            m_oMsgCounter.AddReject ( oPaxosMsg.nodeid() );
            //必须重新进入prepare阶段
            m_bWasRejectBySomeone = true;
            //拒绝该提案节点所附的提案编号
            m_oProposerState.SetOtherProposalID ( oPaxosMsg.rejectbypromiseid() );
        }

        //提案通过
        if ( m_oMsgCounter.IsPassedOnThisRound() )
        {
            int iUseTimeMs = m_oTimeStat.Point();
            BP->GetProposerBP()->AcceptPass ( iUseTimeMs );
            PLGImp ( "[Pass] Start send learn, usetime %dms", iUseTimeMs );
            //退出accept阶段
            ExitAccept();
            //通知所有节点的learner，提案已Accept(但并不保证chosen)，由learner判定
            m_poLearner->ProposerSendSuccess ( GetInstanceID(), m_oProposerState.GetProposalID() );
        }
        //提案未通过
        else if ( m_oMsgCounter.IsRejectedOnThisRound()
                  || m_oMsgCounter.IsAllReceiveOnThisRound() )
        {
            BP->GetProposerBP()->AcceptNotPass();
            PLGImp ( "[Not pass] wait 30ms and Restart prepare" );
            //添加定时器，重新进入Prepare阶段
            AddAcceptTimer ( OtherUtils::FastRand() % 30 + 10 );
        }

        PLGHead ( "END" );
    }
```

**Acceptor**

```cpp
    //
    int Acceptor :: OnPrepare ( const PaxosMsg & oPaxosMsg )
    {
        PLGHead ( "START Msg.InstanceID %lu Msg.from_nodeid %lu Msg.ProposalID %lu",
                  oPaxosMsg.instanceid(), oPaxosMsg.nodeid(), oPaxosMsg.proposalid() );

        BP->GetAcceptorBP()->OnPrepare();

        PaxosMsg oReplyPaxosMsg;
        oReplyPaxosMsg.set_instanceid ( GetInstanceID() );
        oReplyPaxosMsg.set_nodeid ( m_poConfig->GetMyNodeID() );
        oReplyPaxosMsg.set_proposalid ( oPaxosMsg.proposalid() );
        oReplyPaxosMsg.set_msgtype ( MsgType_PaxosPrepareReply );

        BallotNumber oBallot ( oPaxosMsg.proposalid(), oPaxosMsg.nodeid() );

        //新提案编号 >= 当前已接受的提案编号；接受此提案
        if ( oBallot >= m_oAcceptorState.GetPromiseBallot() )
        {
            PLGDebug ( "[Promise] State.PromiseID %lu State.PromiseNodeID %lu "
                       "State.PreAcceptedID %lu State.PreAcceptedNodeID %lu",
                       m_oAcceptorState.GetPromiseBallot().m_llProposalID,
                       m_oAcceptorState.GetPromiseBallot().m_llNodeID,
                       m_oAcceptorState.GetAcceptedBallot().m_llProposalID,
                       m_oAcceptorState.GetAcceptedBallot().m_llNodeID );

            oReplyPaxosMsg.set_preacceptid ( m_oAcceptorState.GetAcceptedBallot().m_llProposalID );
            oReplyPaxosMsg.set_preacceptnodeid ( m_oAcceptorState.GetAcceptedBallot().m_llNodeID );

            //返回当前已接受的提案值
            if ( m_oAcceptorState.GetAcceptedBallot().m_llProposalID > 0 )
            {
                oReplyPaxosMsg.set_value ( m_oAcceptorState.GetAcceptedValue() );
            }

            m_oAcceptorState.SetPromiseBallot ( oBallot );

            int ret = m_oAcceptorState.Persist ( GetInstanceID(), GetLastChecksum() );

            if ( ret != 0 )
            {
                BP->GetAcceptorBP()->OnPreparePersistFail();
                PLGErr ( "Persist fail, Now.InstanceID %lu ret %d",
                         GetInstanceID(), ret );

                return -1;
            }

            BP->GetAcceptorBP()->OnPreparePass();
        }
        else    //已有更新的提案，拒绝此提案
        {
            BP->GetAcceptorBP()->OnPrepareReject();

            PLGDebug ( "[Reject] State.PromiseID %lu State.PromiseNodeID %lu",
                       m_oAcceptorState.GetPromiseBallot().m_llProposalID,
                       m_oAcceptorState.GetPromiseBallot().m_llNodeID );

            oReplyPaxosMsg.set_rejectbypromiseid ( m_oAcceptorState.GetPromiseBallot().m_llProposalID );
        }

        nodeid_t iReplyNodeID = oPaxosMsg.nodeid();

        PLGHead ( "END Now.InstanceID %lu ReplyNodeID %lu",
                  GetInstanceID(), oPaxosMsg.nodeid() );;

        SendMessage ( iReplyNodeID, oReplyPaxosMsg );

        return 0;
    }

    //
    void Acceptor :: OnAccept ( const PaxosMsg & oPaxosMsg )
    {
        PLGHead ( "START Msg.InstanceID %lu Msg.from_nodeid %lu Msg.ProposalID %lu Msg.ValueLen %zu",
                  oPaxosMsg.instanceid(), oPaxosMsg.nodeid(), oPaxosMsg.proposalid(), oPaxosMsg.value().size() );

        BP->GetAcceptorBP()->OnAccept();

        PaxosMsg oReplyPaxosMsg;
        oReplyPaxosMsg.set_instanceid ( GetInstanceID() );
        oReplyPaxosMsg.set_nodeid ( m_poConfig->GetMyNodeID() );
        oReplyPaxosMsg.set_proposalid ( oPaxosMsg.proposalid() );
        oReplyPaxosMsg.set_msgtype ( MsgType_PaxosAcceptReply );

        BallotNumber oBallot ( oPaxosMsg.proposalid(), oPaxosMsg.nodeid() );

        //新提案编号 >= 当前已接受的提案编号；接受此提案
        if ( oBallot >= m_oAcceptorState.GetPromiseBallot() )
        {
            PLGDebug ( "[Promise] State.PromiseID %lu State.PromiseNodeID %lu "
                       "State.PreAcceptedID %lu State.PreAcceptedNodeID %lu",
                       m_oAcceptorState.GetPromiseBallot().m_llProposalID,
                       m_oAcceptorState.GetPromiseBallot().m_llNodeID,
                       m_oAcceptorState.GetAcceptedBallot().m_llProposalID,
                       m_oAcceptorState.GetAcceptedBallot().m_llNodeID );

            m_oAcceptorState.SetPromiseBallot ( oBallot );
            m_oAcceptorState.SetAcceptedBallot ( oBallot );
            m_oAcceptorState.SetAcceptedValue ( oPaxosMsg.value() );

            int ret = m_oAcceptorState.Persist ( GetInstanceID(), GetLastChecksum() );

            if ( ret != 0 )
            {
                BP->GetAcceptorBP()->OnAcceptPersistFail();

                PLGErr ( "Persist fail, Now.InstanceID %lu ret %d",
                         GetInstanceID(), ret );

                return;
            }

            BP->GetAcceptorBP()->OnAcceptPass();
        }
        else        //已有更新的提案，拒绝此提案
        {
            BP->GetAcceptorBP()->OnAcceptReject();

            PLGDebug ( "[Reject] State.PromiseID %lu State.PromiseNodeID %lu",
                       m_oAcceptorState.GetPromiseBallot().m_llProposalID,
                       m_oAcceptorState.GetPromiseBallot().m_llNodeID );

            oReplyPaxosMsg.set_rejectbypromiseid ( m_oAcceptorState.GetPromiseBallot().m_llProposalID );
        }

        nodeid_t iReplyNodeID = oPaxosMsg.nodeid();

        PLGHead ( "END Now.InstanceID %lu ReplyNodeID %lu",
                  GetInstanceID(), oPaxosMsg.nodeid() );

        SendMessage ( iReplyNodeID, oReplyPaxosMsg );
    }
```

**Learner**

Learner定时发送当前的Instance Id，尝试习得自该Instance Id后的值，处理逻辑如下：

```cpp
    void Learner :: OnAskforLearn ( const PaxosMsg & oPaxosMsg )
    {
        BP->GetLearnerBP()->OnAskforLearn();

        PLGHead ( "START Msg.InstanceID %lu Now.InstanceID %lu Msg.from_nodeid %lu MinChosenInstanceID %lu",
                  oPaxosMsg.instanceid(), GetInstanceID(), oPaxosMsg.nodeid(),
                  m_poCheckpointMgr->GetMinChosenInstanceID() );

        SetSeenInstanceID ( oPaxosMsg.instanceid(), oPaxosMsg.nodeid() );

        //发现一个新的Follower 
        if ( oPaxosMsg.proposalnodeid() == m_poConfig->GetMyNodeID() )
        {
            //Found a node follow me.
            PLImp ( "Found a node %lu follow me.", oPaxosMsg.nodeid() );
            m_poConfig->AddFollowerNode ( oPaxosMsg.nodeid() );
        }

        //需要习得的Instance Id比本节点的更新，无法从本节点习得，直接返回
        if ( oPaxosMsg.instanceid() >= GetInstanceID() )
        {
            return;
        }

        //需要习得的Instance Id尚未被归档(Checkpoint)，可以从本节点习得
        if ( oPaxosMsg.instanceid() >= m_poCheckpointMgr->GetMinChosenInstanceID() )
        {
            //向Learner
            if ( !m_oLearnerSender.Prepare ( oPaxosMsg.instanceid(), oPaxosMsg.nodeid() ) )
            {
                BP->GetLearnerBP()->OnAskforLearnGetLockFail();

                PLGErr ( "LearnerSender working for others." );

                if ( oPaxosMsg.instanceid() == ( GetInstanceID() - 1 ) )
                {
                    PLGImp ( "InstanceID only difference one, just send this value to other." );
                    //send one value
                    AcceptorStateData oState;
                    int ret = m_oPaxosLog.ReadState ( m_poConfig->GetMyGroupIdx(), oPaxosMsg.instanceid(), oState );

                    if ( ret == 0 )
                    {
                        BallotNumber oBallot ( oState.acceptedid(), oState.acceptednodeid() );
                        SendLearnValue ( oPaxosMsg.nodeid(), oPaxosMsg.instanceid(), oBallot, oState.acceptedvalue(), 0, false );
                    }
                }

                return;
            }
        }

        //已经归档，发送本节点当前的Instance Id信息，交由Learner的发起者决定是否决定是否做做归档数据对齐
        SendNowInstanceID ( oPaxosMsg.instanceid(), oPaxosMsg.nodeid() );
    }
```

## 质量属性

### 可靠性

### 性能



