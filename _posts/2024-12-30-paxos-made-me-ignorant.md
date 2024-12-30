---
title: "Paxos一知半解"
---


在2024年的末尾，忽然发现自己今年还没写博客。不是不想写，主要是越发觉得自己所知甚少，怕写出来的东西漏洞百出，让人耻笑。但是自己平时也没少做笔记，琢磨着光做笔记不总结也不是回事。便想了几个主题——Concurrency Programming、MongoDB、Kubernetes，想着挑一个分享一下。结果前两天看Paxos论文走火入了魔，没时间搞其他东西了。索性就把这两天自己总结的Paxos笔记当作2024年的唯一一篇博客，以纪念我逝去的青春。

至于你问我为什么走火入魔，一个是我不聪明，另一个是Basic Paxos和Fast Paxos两篇论文确实有点晦涩难懂，特别是Fast Paxos。并且论文语言太简练，有很多隐藏的点并没有直接挑明，需要自己寻思。并且在翻看论文过程中需要不断回忆或者向前查阅单词的出处或定义。这两天研究下来，最大的感悟就是自己确实不是做科研的料，这些东西让我想破脑袋也想不出来。更难受的是，我连这两篇论文都没有彻底看懂，只留下了这篇一知半解的笔记。


# 1 classic paxos/ basic paxos

classic paxos 通过多数派读写来最终确定一个共识Value。这里要注意，达成共识并不是让所有节点都达到一致的状态，跟Raft类似，提交日志并不意味着把log复制到所有节点上。Value也不一定是Proposer自己决定的，可能是修复前一轮未完成的Paxos，跟Raft类似，leader也会提交前任leader的日志。

有三个角色：Proposer、Acceptor、Learner。

## 1.1 目标

**终极目标**：
- safety requirements
	- 只有一个proposed value会被chosen
	- 其他人可以learn被chosen的value
- liveness requirements（两个eventually）
	- ensure that some proposed value is eventually chosen
	- if a value has been chosen, then a process can eventually learn the value.

使用少数服从多数的方式达成共识。为了让所有acceptor都参与进来，以及保证只有一个proposed value时也能达成共识，提出了P1：An acceptor must accept the first proposal that it receives. 

> 终极目标到充分目标A的论证：
> 
> 那么对于non-first proposal，acceptor能不能接受呢？答案是肯定的。如果不允许acceptor接受non-first proposal，一旦第一轮没有形成多数派，那么后面就无法再达成共识，违反了liveness requirements。
> 
> 如果允许acceptor接受多个proposal，那么就可能有多个proposal被选中。

由此产生了目标A：
- **可以选中多个proposal达成共识，但是被选中的所有proposal应该具有相同的value。**

目标A的充分目标P2：
- **每个proposal包含一个number和value**，不同proposal的number不同，value可以相同。numbers are totally ordered。
- **P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v .**

> 充分目标P2到它的充分目标P2a的论证：
> 
> 一个proposal 要被选中，至少要被大多数acceptor接受。所以只要大多数acceptor不接受不符合预期的proposal就可以。
> 但是“大多数”是一个虚词，无法精确做到。所以可以要求每个acceptor不接受不符合预期的proposal。

由此得到充分目标P2的充分目标P2a：If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v .

> P2a到P2b的演进：
> 
> P2a和P1是有冲突的：如果一个acceptor因为某种原因（比如crash）没有接受过任何proposal，而一个proposer提出了一个有不同value，且有更高number的proposal。按照P1，这个acceptor应该接受这个proposal，按照P2，这个acceptor不应该接受这个proposal。

由此得到充分目标P2的加强版充分目标P2b：If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v .

所以实现了P2b，也就实现了P2a，也就实现了P2。

还有P2b到P2c的推导，可参见论文——Paxos Made Simple。

## 1.2 适用场景

异步消息通信系统（asynchronous message passing system）

非拜占庭错误。

过半数的acceptor存活。

## 1.3 实现

Acceptor是**系统中唯一一个必须持久化数据的角色**(Learner和Proposer都可以不持久化数据)。至少需要记住：
- highest-numbered proposal that it has ever accepted
- the number of the highest-numbered prepare request to which it has responded

**Accptor之间并不通信**，如果想让其它未存储最终决议的Accptors获取最终决议，需要Proposer重新发起proposal。

整个过程分为两个阶段：Phase 1和Phase 2。

1. Phase 1
    1. Proposer选择一个提案编号N（全局唯一）并发送Prepare(N)请求给至少大多数Acceptor。
    2. Acceptor收到Prepare(N)消息，如果N大于该Acceptor已经承诺的proposal的最大编号（也就是说对于同一个proposal number，一个Acceptor只会回复一次），那么
	    1. 它会记住这个N，并承诺不会accept任何小于N的proposal。
	    2. 并且回复Promise(N, {N', V'})，{N', V'}是此Acceptor接受过的具有最大编号的proposal。如果还没有接受过任何proposal，V'就是null。
2. Phase 2
    1. 如果Proposer收到半数以上Acceptor对其Prepare(N)请求的响应，它就会发送Accept({N, V})请求给这些Acceptors。V的取值有两种：如果所有Promise响应中V'都是null，那么由Proposer自主决定V的值，否则取所有Promise中具有最大N'的V'。
    2. Acceptor收到Accept({N, V})请求之后，只要该Acceptor没有对编号大于N的Prepare请求做出过承诺，并且没有接受过编号为N的proposal，就接受该提案。（注意，接收到Prepare请求的大多数和接收到Accept请求的大多数可以不是相同的大多数，所以不要直接判断N是否等于承诺的最大编号，来决定是否接受proposal）
	    1. “没有接受过编号为N的proposal”这个条件是fast-paxos论文新增的。加不加其实都可以，因为Phase 1会保证对同一个proposal number只承诺一次。


![一个简单的示例](/assets/images/paxos-two-phase-basic-workflow.png)


### 1.3.1 learner

一个简单直接的Learner的实现是，获取所有Acceptor接受过的提议，然后看哪个提议被多数的Acceptor接受，那么该提议对应的值就是被选定的。

另外，也可以把Learner看作一个Proposer，根据协议流程，发起一个正常的proposal。

### 1.3.2 Proposer的两种行为：写入和修复

Phase 1的Acceptor的响应有两层含义：
1. promise。承诺不会接受来自“过去”的写入，过去意味着过时。要不然一直有过去的写入，proposer无法learn到过去accepted了什么值。
2. learn。返回过去的写入，让Proposer不破坏之前达成的共识。（虽然过去的写入并不一定被大多数Acceptors共识，但对于单个Acceptor来说，它并不知道）

如果过去已经达成共识，那么Proposer提出的proposal的value一定是那个共识value，称这个行为是修复。

如果过去没有任何写入，那么Proposer提出的proposal的value可以由Proposer自行决定，称这个行为为写入。

如果过去虽然对acceptors有写入，但是没有达成共识，那么Proposer的行为取决于Proposer收到了哪些acceptors的响应。比如下面情况，proposer1只写入acceptor1后就退出，共识虽然没有达成，proposer2再次运行，联系到了acceptor1，也只能接着写入没有达成共识的v。
这种现象在raft中也有，切主时，新主上如果有些log没有提交，也会继续把这些log提交（会随着新主的日志提交）。

对于Paxos来说，如果想写入新的值，只能新开一个Paxos run。

![](/assets/images/paxos-write-uncertained-value.png)


### 1.3.3 acceptor的安全保证

原子性

并发安全

响应之前先写stable storage

### 1.3.4 读取

在Basic Paxos协议中，对于决议（value）的读取也是需要联系大多数Acceptor的。

## 1.4 常见问题

### 1.4.1 proposal为什么需要一个total ordered number

acceptor需要能接受多个proposal。（否则如果第一轮没有达成共识，就再也达不成了）

多个proposal需要被区分开。

有了 total ordered之后，acceptor可以对proposal做排序，也就能知道哪些是“过去的”proposal。

一个简单的Proposal Number定义（来自braft）：
![](/assets/images/paxos-proposal-number-example-braft.png)

**那么不同的proposal需要有全局唯一的编号吗？不需要。** 

假设两个不同的proposer同时提出了两个具有相同编号的proposal，也只有一个proposal能获得大多数Acceptors的promise。因为Acceptor对于同一个编号只会回复一次promise，先到先得。

就像Raft的leader election，可以有两个candidate发起相同term的RequestVote，但是follower对同一个term只会投票一次，确保了一个term只有一个leader。

所以Acceptor需要持久化“它已经回复的所有proposal的最大编号”，Raft的follower需要持久化“它投过的最大term”。

所以，proposer在生成proposal number的时候，不需要保证全局唯一，只要能保证已知唯一就好了。

在fast-paxos论文中，proposal number有了一个更正式的名字——round（简称做rnd），相同的proposal number属于同一个round。

### 1.4.2 为什么需要两轮多数派交互

至少需要两轮交互——学习和写入。

所以“为什么需要两轮多数派交互？”这个问题，可以改成“为什么在写入前需要学习？”。

如果没有了学习那一轮，那么无法保证多个proposal有相同的value。比如proposal1已经被accepted了，后来的proposal2也需要有相同的value。

这也是论文中“Learning about proposals already accepted is easy enough; predicting future acceptances is hard.”的意思，因为无法做到预测未来，所以需要在写入前了解过去有哪个value被chosen，来保证多个proposal有相同的value。

如果Phase 1中大多数Acceptors返回的value都是null，那么proposer可以自主选择value，就可以进入fast-paxos。

### 1.4.3 如何理解proposer学习到一个未达成共识value的行为

因为phase1 也仅要求请求发送给多数派acceptor，所以一旦proposer收到了value，就意味着这个value可能已经达成共识——返回value的这个acceptor和剩余的少数派组成多数派。那么，为了安全，proposer就不能自己想写入什么就写入什么了。

比如有三个acceptor——a，b和c，proposer1仅写入a之后就退出了。proposer2的prepare 请求发送给了a和b，proposer2收到a的value之后，判断这个value可能已经被a和c接受而达成共识，所以proposer1后面提出的proposal是proposer1写入的value。

如果phase1的prepare请求发送给足够多的节点（至少是majority，最多是全部节点），了解到目前没有值达成共识，那么proposer也可以提出具有任意值的proposal。


# 2 Fast-Paxos

## 2.1 重新审视Classic-Paxos

Classic-Paxos使用若干round（至少一round）来对一个value达成共识。每round由两个阶段组成：Phase 1和Phase 2。

每个Phase都请求大多数（majority）Acceptor。所以不同round的Phase 1和Phase 2是可以互相感知的，因为两个majority一定有交集，也就保证已经达成共识的value在后续round中依然是共识。

quorum选择majority的原因是：
- 在不同的phase，对相同的round number，每个acceptor只会响应一次。所以在Phase 1，即使两个proposal number相同，只会有一个proposer获得majority，也就是说只有一个proposer会获得这个 round的写入权。
- 最大化acceptor的fault-tolerance。

如果每轮round使用不同的quorum会怎么样呢？这里我们把round i的quorum称作i-quorum。

## 2.2 Fast-Paxos流程

Fast-Paxos中有两种round——fast round和classic round。fast round和classic round只有Phase 2是不同的，对于fast round，
- 如果Phase 2是一个写入行为（没有value可以选，需要 coordinator提出value），那么coordinator会给acceptors发送any message。
- acceptors如果收到any message，后续会直接接受第一个proposer 提出的round i的proposal value。

所以对于fast round来说，不同的acceptors可能接受不同的value，打破了classic paxos中的约束——只有一个proposer会获得这个 round的写入权。（但还是会保留一个acceptor在一个round只会被写一次（或者叫vote for）的原则）

这个约束的打破是很严重的，需要重新思考整个算法的安全性。

> 此时有两种quorum：fast round quorum和classic round quorum。

fast-paxos论文经过推导，得到了保证safety的**Quorum Requirement**：
1. 任意两个quorum都需要有交集；
2. 称任意两个fast round quorum的交集为S，S和任意的classic round quorum要有交集；

条件1是比较明显的，从classic paxos一脉相承，是一个最基础的保证。

条件2是怎么意思呢？是为了保证在fast round达成共识的value，一定能被classic round选出来。假设在fast round有两个proposer向acceptors写入value，最终有一个proposer达到了fast round quorum。
- 两个fast round quorum的交集里面的value一定是达成共识的value，且只有一个。
- 这个交集要和任意的classic round quorum都有交集。意味无论classic round的两个Phase怎么挑选quorum数量的acceptors，里面的大多数acceptors都在fast round达成了共识，有着相同的value。

那么两个quorum该如何取值呢？假设acceptor数量固定为N，根据Quorum Requirement写入几个不等式：
1.  f-quorum + f-quorum > N （Requirement 1）
2.  f-quorum + c-quorum > N （Requirement 1）
3.  c-quorum + c-quorum > N （Requirement 1）
4.  (f-quorum + f-quorum - N) + c-quorum > N  （Requirement 2）
5. 假设f-quorum >= c-quorum，如果f-quorum < c-quorum就交换两者的值

简化得到：
1.  f-quorum >= c-quorum > N/2
2.  `2*f-quorum + c-quorum > 2N`

如果希望f-quorum最小（fast round 的fault-tolerance更好），那么`f-quorum = c-quorum = N*2/3+1`。也就是说，f-quorum最小是`N*2/3+1`，这种情况下，c-quorum最小是`N*2/3+1`。
如果希望c-quorum最小（classic round 的fault-tolerance更好），那么`c-quorum = N/2+1, f-quorum = N*3/4+1`。也就是说，c-quorum最小是`N/2+1`，这种情况下，f-quorum最小是`N*3/4+1`。

## 2.3 比classic paxos快在哪

classic paxos需要6个或者7个消息延迟：client -> proposer -> Phase 1 &&  Phase 2 -> acceptor -> learner -> client. 

![](/assets/images/paxos-classic-roundtrip-example.png)

fast paxos正常情况下需要四个消息延迟：client -> acceptor -> coordinator  -> learner -> client. 

![](/assets/images/paxos-fast-roundtrip-example.png)


# 参考

- 《微信 PaxosStore：深入浅出 Paxos 算法协议》 https://www.infoq.cn/article/wechat-paxosstore-paxos-algorithm-protocol/ 
- 《可靠分布式系统-paxos的直观解释》 https://blog.openacid.com/algo/paxos/ 
- 《Paxos——凤凰架构》 https://icyfenix.cn/distribution/consensus/paxos.html 
-  《一个做的比较好的PPT》 https://ongardie.net/static/raft/userstudy/paxos.pdf
- 《braft讲解的paxos》 https://github.com/baidu/braft/blob/master/docs/cn/paxos_protocol.md
- 《Paxos 后传——小岛历史的延续》 https://blog.howardlau.me/programming/paxos-optimizations.html
