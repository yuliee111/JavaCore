<h3 style="padding-bottom:6px; padding-left:20px; color:#ffffff; background-color:#E74C3C;">一、2PC协议</h3>

> 2PC 是 *Two-Phase Commit Protocol* 的缩写，中文名称 **两阶段提交协议**，它是为了解决 *分布式事务一致性* 问题而产生的算法。分布式一致性问题是分布式系统各个节点（分区）如何就一项决议达成一致的问题，构成的一组操作称为**事务** （又称 **分布式事务**，**全局事务**），各个节点的事务称为**分支事务**。**两阶段提交协议** 是一种原子提交协议、共识协议。

**两阶段提交协议** 是指事务提交的过程分为两个阶段：**准备阶段** 和 **提交阶段**。其中事务的发起者称为 **事务协调者** ，事务的执行者称为 **参与者**。一般，在一个事务中有一个事务协调者和多个参与者。

![2PC](https://i.loli.net/2019/01/16/5c3f3a888ca74.png)

#### 阶段一：准备阶段：

又称 投票阶段。首先，协调者询问全局事务中各个参与者是否可以执行操作。然后，全局事务中的各个参与者执行操作（占有资源）并做出回答 YES/NO 给协调者；

#### 阶段二：提交阶段：

又称 完成阶段。最后，协调者检查事务中的各个参与者的反馈。

**成功:** 在规定的时效内全部反馈 YES，则协调器向所有参与者发送事务提交消息。每个参与者完成操作，并释放事务期间持有的所有锁和资源。每个参与者向协调器发送一个确认。当事务协调器收到所有确认时，协调器完成事务。

**失败:** 在规定的时效内有参与者未对阶段一做回复（超时）或有参与者回复 NO，则协调器向所有参与者发送事务回滚消息。每个参与者使用撤消日志撤消事务，并释放事务期间保留的资源和锁。每个参与者向协调器发送一个确认。当收到所有确认时，协调器将撤消事务。



#### 下面通过一个案例来说明：

**背景**：比如某用户想从杭州去巴黎，但杭州没有直达巴黎的机票，他想先从杭州转到北京，然后从北京飞往巴黎。

他想通过携程APP进行购买两张票，要么两张票都购买成功，要么两张票都购买失败。

**过程**：首先用户先通过携程选择杭州到北京、北京到巴黎的机票；如果两个航班都有机票话，然后下单购买两个航班的机票，从杭州到北京的机票是首都航空公司提供，从北京到巴黎的机票是法国航空公司提供。

**分析**：整个购买两张机票的过程就是一个事务，分别购买两个班次的行为就是两个操作。

准备阶段：选择航班机票时就是准备阶段，被选择的航空公司票务系统就是参与者，通过选择机票就是占有资源(占有资源是有时间限制的)；

![准备阶段](https://i.loli.net/2019/01/16/5c3f3e5d530ae.png)

消息反馈：

![反馈](https://i.loli.net/2019/01/16/5c3f3e94d2bdd.png)

提交阶段：携程作为协调者会查看参与者(各航空公司票务系统)是否可以提供机票，如果都是YES，表示可以提交订单，否则有NO，只能放弃操作(事务回滚)，也是放弃对资源的占有。

![提交阶段](https://i.loli.net/2019/01/16/5c3f40464dec3.png)



#### 2PC缺点：

**同步阻塞**：我们前面讲过，2PC协议就是为了解决分布式事务一致性问题的，而可用性与一致性往往存在矛盾。假设分区节点之间出现网络通信问题，那么消息就不能及时传递给响应的节点，为了到达一致性，整个分布式系统(准确地将是参与事务的机器组成的系统)将会陷入阻塞状态，所以2PC协议是一个典型的 **阻塞式协议**。

**单点问题**：如果协调者出现故障，参与者将一直处在等待状态(等待协调者发送消息指令)，分支事务占有资源也不能释放，直到收到提交/回滚命令为止。这就会协调者的单点问题。

**脑裂问题**：在提交阶段中，只有部分参与者接收到协调者发送的 `Commit` 指令，则会出现数据不一致的问题。



**2PC如何解决故障问题？**

> 两阶段提交协议的参与者使用协议状态的日志记录。协议的恢复过程使用日志记录，这些记录通常生成速度很慢，但在失败后仍然存在。存在许多协议变体，它们主要在日志策略和恢复机制方面有所不同。尽管恢复过程通常很少使用，但由于协议要考虑和支持许多可能的故障场景，因此它构成了协议的一个重要部分。
>
> 协议的工作方式如下：一个节点是指定的协调器，它是主站点，网络中的其余节点被指定为参与者。协议假定每个节点上都有一个稳定的存储，有一个提前写入日志，不会永远崩溃，提前写入日志中的数据不会在崩溃中丢失或损坏，并且任何两个节点都可以相互通信。最后一个假设没有太多限制，因为网络通信通常可以被重新路由。前两个假设要强大得多；如果一个节点被完全破坏，那么数据可能会丢失。

> 在到达事务的最后一步之后，协调器将启动该协议。然后，参与者根据是否在参与者处成功处理了事务，使用协议消息或中止消息进行响应。



#### 2PC与XA事务

X/Open 的XA事务就是使用的2PC协议，事务的协调者是TM（事务管理器），事务的参与者是TM（资源管理器）。为了解决2PC中事务协调者的单点问题，又出于性能和可靠性的原因，协调者的角色可以转移到另一个TM（事务管理器）。参与者之间不交换2PC信息，而是与各自的TM（事务管理器）交换信息。相关的TM（事务管理器）相互通信以执行上面的2PC协议模式，“代表”各自的参与者，以终止该事务。在这种体系结构中，协议是完全分布式的（不需要任何中央处理组件或数据结构），并且可以有效地随着网络节点的数量（网络大小）而扩展。



#### 树型两阶段提交协议

The TreeTwo-phase Commit Protocol (树形两阶段提交，T2PC，t2pc，Tree-2PC）也称为嵌套2PC或递归2PC，是2PC的常见变体协议，它更好地利用了底层通信基础设施。分布式事务中的参与者通常按照定义树结构的顺序调用，即调用树，其中参与者是节点，边缘是调用（通信链接）。通常使用同一个树通过2PC协议完成事务，但原则上也可以使用另一个通信树来完成事务。在 Tree-2PC 中，协调器被认为是通信树（倒挂树）的根（“top”），而参与者是其他节点。协调器可以是发起事务的节点（递归地（传递地）调用其他参与者），但同一树中的另一个节点可以代替协调器角色。来自协调器的2PC消息被“向下”传播到树上，而发送到协调器的消息则被一个参与者从树下的所有参与者“收集”到，然后在树上发送适当的消息“向上”（除中止消息外，该消息在接收到该消息后立即被“向上”传播，或者如果当前参与者启动了博尔特）



#### 动态两阶段提交

The Dynamic Two-phase Commit Protocol (动态两阶段提交，D2PC，d2pc，Dynamic-2PC）是没有预定协调器的 树形两阶段提交协议 的变体协议。协议消息（是投票）从所有叶开始传播，每个叶在代表事务完成其任务时（准备就绪）。中间（非叶）节点在协议消息发送到最后一个（单个）相邻节点（尚未从该节点接收到协议消息）时发送就绪。协调器是通过在事务树上的冲突位置对协议消息进行竞速来动态确定的。它们要么在事务树节点上发生冲突，要么作为协调器，要么在树边缘发生冲突。在后一种情况下，两个边缘的节点之一被选为协调器（任何节点）。D2PC是时间最优的（在特定事务树的所有实例和任何特定 Tree-2PC 协议实现中；所有实例都有相同的树；每个实例都有不同的节点作为协调器）：通过选择最佳协调器，D2PC在尽可能短的时间内提交协调器和每个参与者，从而允许在每个事务参与者（树节点）中尽早释放锁定的资源。



---

<h3 style="padding-bottom:6px; padding-left:20px; color:#ffffff; background-color:#E74C3C;">二、3PC协议</h3>

> 3PC 是 *Three-Phase Commit Protocol* 的缩写，中文名称 **三阶段提交协议**，与两阶段提交协议（2PC）不同，3PC是非阻塞协议。具体来说，3PC对事务提交或中止之前所需的时间量设置了一个上限。此属性确保如果给定的事务正试图通过3PC提交并持有一些资源锁，则它将在超时后释放这些锁。
>
> 在原来2PC协议的基础上，将2阶段扩展到3阶段：**询问提交阶段**、**预提交阶段**、**执行提交阶段**。

![3PC](https://i.loli.net/2019/01/16/5c3f0b5b4af27.png)

#### 阶段一：询问提交阶段（CanCommit）：

这个阶段（CanCommit）和2PC的准备阶段非常相似。事务协调者接受事务请求。如果出现协调者出现故障，则中止事务（即使在恢复时，它将认为事务已中止）。否则继续做以下操作：

​	**1.事务询问**：事务中的协调者向事务中的所有参与者发送消息，询问各个参与者是否可以提交事务；

​	**2.事务反馈**：各个参与者根据当前资源判断是否可以提交事务（一般会占有资源），并反馈给协调者。



#### 阶段二：预提交阶段（PreCommit）：

事务中的协调者在收到各个参与者反馈后，做出判断，出现以下两种情况：

**1.** 如果事务中的所有参与者都表示可以提交事务，则做执行 **事务预提交** 

​	**1.1.发送预提交请求**：事务中的协调者向各个参与者发送预提交(PreCommit)指令，并进入已准备(prepared)状态；

​	**1.2.事务预提交**：事务中的参与者接受到协调者的预提交(PreCommit)指令，会执行事务操作，并将undo和redo信息记录到事务日志中。（undo和redo分别表示撤销和恢复）；

​	**1.3.反馈事务预提交的结果**：在参与者成功的执行了事务操作，并向协调者返回ACK响应，如果协调者出现故障，那么参与者将不断的重试发送ACK。之后开始等待最终事务提交指令。

**2.** 如果其中之一的参与者表示不能提交事务或超时反馈，则做 **中断事务** 处理

​	**2.1.发送中断事务指令**：事务中的协调者向各个参与者发送中断指令(abort)；

​	**2.2.中断事务**：各个参与者就收到中断指令或超时未收到任何指令时，会做中断事务处理。



#### 阶段三：执行提交阶段（DoCommit）：

事务中的协调者在先接收各个参与者的反馈，然后做出判断，也会出现以下两种情况：

**1.** 如果事务中的参与者成功执行的预提交事务操作，并返回成功ACK，则 **执行事务提交**：

​	**1.1.发送执行事务提交指令**：事务协调者发送执行事务提交指令给各个参与者；

​	**1.2.事务提交**：各个参与者收到协调者发送的执行事务提交的指令，然后执行事务提交；

​	**1.3.反馈事务提交结果**：各个参与者执行完事务提交操作后，反馈结果给事务协调者；

​	**1.4.完成事务**：协调者接收到各个参与者执行提交的反馈，事务完成。



**2.** 如果事务中的参与者执行预提交事务操作失败或超时反馈，则 **中断事务**：

​	**1.1.发送中断事务指令**：事务中的协调者向各个参与者发送中断指令(abort)；

​	**1.2.事务回滚**：各个参与者收到协调者发送的执行事务回滚的指令，然后执行事务回滚；

​	**1.3.反馈事务回滚结果**：各个参与者执行完事务回滚操作后，反馈结果给事务协调者；

​	**1.4.中断事务**：协调者接收到各个参与者执行回滚的反馈，事务中断。



**说明**：执行提交阶段（DoCommit）时，可能协调者或网络出现问题，都有可能导致事务的部分参与者无法接收到协调者的指令（包括执行提交和中断事务），在这种情况下，部分参与者会在超时等待之后继续进行事务提交。有一定小概率会指定数据不一致的情况，所以这也是3PC协议的一大缺点。



参考资料：

[WIKIPEDIA：Two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)