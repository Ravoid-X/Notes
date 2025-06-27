## 概述
1. 少数在工程实践中证实的强一致性、高可用的去中心化分布式协议。
2. Paxos 协议的流程较为复杂，其基本思想类似于人类社会的投票过程。
3. Paxos 协议中，有一组完全对等的参与节点（accpetor），各自就某一事件做出决议，如果某个决议获得了超过半数节点的同意则生效。
4. Paxos 协议中只要有超过一半的节点正常，就可以工作，能很好对抗宕机、网络分化等异常情况。

## 概念
1. Proposer：提案者，可以有多个，提出提案 (Proposal)。Proposal 信息包括提案编号 (Proposal ID) 和提议 (Value)。。
2. value：在工程中可以是任何操作，例如“修改某个变量的值为某个值”、“设置当前 primary 为某个节点”等等。
3. 不同的 Proposer 可以提出不同的甚至矛盾的 value，但对同一轮 Paxos 过程，最多只有一个 value 被批准。
4. Acceptor：批准者，有 N 个，value 必须获得超过半数($\frac N 2 +1$)的 Acceptor 批准后才能通过。Acceptor 之间完全对等独立。
5. Learner：学习者，Learner 学习被批准的 value。通过读取各个 Proposer 对 value 的选择结果，如果某个 value 被超过半数 Proposer 通过，则 Learner 学习到了这个 value。可用于扩展读性能，或者跨区域之间读操作。
6. 上述三类角色只是逻辑上的划分，实践中一个节点可以同时充当这三类角色。

## 流程
1. Paxos 协议一轮一轮的进行，每轮都有一个编号。
2. 如果某一轮 Paxos 协议批准了某个 value，则以后各轮 Paxos 只能批准这个 value。
3. 一次 Paxos 协议实例只能批准一个 value，这也是 Paxos 协议强一致性的重要体现。
4. 每轮 Paxos 协议分为准备、批准和学习阶段\
（1）Prepare 阶段。Proposer 向 Acceptors 发出 Prepare 请求，Acceptors 针对收到的 Prepare 请求进行 Promise 承诺。\
（2）Accept 阶段。Proposer 收到多数 Acceptors 承诺的 Promise 后，向 Acceptors 发出 Propose 请求，Acceptors 针对收到的 Propose 请求进行 Accept 处理。\
（3）Learn 阶段。Proposer 在收到多数 Acceptors 的 Accept 之后，标志着本次 Accept成功，决议形成，将形成的决议发送给所有 Learners。
5. Prepare 请求: Proposer 生成全局唯一且递增的 Proposal ID (可使用时间戳加 Server ID)，向所有 Acceptors 发送。
6. Promise 承诺：Acceptors 收到 Prepare 请求后，做出“两个承诺，一个应答”\
（1）不再接受 Proposal ID 小于等于当前请求的 Prepare 请求。\
（2）不再接受 Proposal ID 小于当前请求的 Propose 请求。\
（3）不违背以前作出的承诺下，回复已经 Accept 过的提案中 Proposal ID 最大的那个提案的 Value 和 Proposal ID，没有则返回空值
7. Propose: Proposer 收到多数 Acceptors 的 Promise 应答后，从应答中选择 Proposal ID 最大的提案的 Value，作为本次要发起的提案。如果所有应答的提案 Value 均为空值，则可以自己随意决定提案Value。然后携带当前 Proposal ID，向所有 Acceptors 发送 Propose 请求。
8. Accept: Acceptor 收到 Propose 请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID 和提案 Value。
9. Learn: Proposer 收到多数 Acceptors 的 Accept 后，决议形成，将形成的决议发送给所有 Learners。