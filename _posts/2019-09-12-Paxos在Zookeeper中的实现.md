---
layout: default
title: Paxos在Zookeeper中的实现
---
## 一，Paxos的理解<br>
#### 1， The Part-Time Parliament<br>
[参考phpaxos](https://zhuanlan.zhihu.com/p/21438357)<br>
[The Part-Time Parliament](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/The-Part-Time-Parliament.pdf)<br>
[译文](https://www.cnblogs.com/hzmark/p/The_Part-Time_Parliament.html)<br>
<br>
主要是  论文里的数学定义**2.1 Mathematical Results**<br>
$\beta$: 任意的表决集合<br>
$B$: 一轮投票参与元素的集合<br>
$B_{bal}$这轮投票所使用的编号,表示第几轮投票<br>
MaxVote(b,Q,β):为Q中编号小于b的最大的投票<br>
$B_{dec}$ 这轮投票的目标提议（可以为任何可描述的事物，比如一串二进制，一个图像等）<br>
$B_{qrm}$ 一轮 成功的投票所需要的投票者集合（可理解为多数派）<br>
$B_{vot}$这轮投票实际投票者集合（实际真正投票成功的投票者）<br>
<br>
###### 投票约束<br>
$B1(\beta)$:$\beta$中的每一轮投票，编号必须是唯一的。<br>
$B2(\beta)$:$\beta$中任意两轮投票的法定人数集合，至少有一个重合的议员。可以理解为只有多数派进行了投票，本轮投票才算成功.<br>
$B3(\beta)$:对于$\beta$中的一轮投票编号为$B_{bal}$的投票， 假如多数派中存在任意一名成员投过比$B_{bal}$编号小的票$B'$， 那么$B_{dec}=B'_{dec}$<br>
<br>
可以用数学符号来描述以上约束：<br>
<br>
<br>
$$
\begin{aligned}
& B1(\beta) \triangleq \forall B, B' \in \beta :  (B \neq B')\Rarr (B_{bal}\neq B'_{bal}) \\
& B2(\beta) \triangleq \forall B, B' \in \beta :  B_{qrm} \cap B'_{qrm}=\empty \\
& B3(\beta) \triangleq∀B ∈ \beta : (MaxVote(B_{bal}, B_{qrm}, \beta)_{bal} = -\infin) \Rarr \\
 & \mskip{4cm} (B_{dec} = MaxVote(B_{bal}, B_{qrm}, \beta)dec)
\end{aligned}
$$
<br>
#### 2，Paxos Made Simple<br>
[Paxos Made Simple](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/paxos-simple-Copy.pdf)<br>
[译文](https://blog.csdn.net/fantasy0126/article/details/73698660)<br>
[维基百科上的中文](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)<br>
主要是**2.2Choosing a Value**<br>

* P1：一个 acceptor 必须接受（accept）第一次收到的提案。<br>
* P2：一旦一个具有 value v 的提案被批准（chosen），那么之后批准（chosen）的提案必须具有 value v。<br>
&ensp; * P2$^a$.一旦一个具有 value v 的提案被批准（chosen），那么之后任何 acceptor 再次接受（accept）的提案必须具有 value v。<br>
&ensp; * P2$^b$. 一旦一个具有 value v 的提案被批准（chosen），那么以后任何 proposer 提出的提案必须具有 value v。<br>
&ensp; * P2$^c$：如果一个编号为 n 的提案具有 value v，那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于 n 的任何提案，要么他们已经接受（accept）的所有编号小于 n 的提案中编号最大的那个提案具有 value v。<br>

 通过一个决议分为两个阶段：<br>

* prepare阶段：<br>
proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；<br>
acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息(回复消息表示接受accept)，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；<br>
* 批准阶段：<br>
当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和根据P2c决定的value（如果根据P2c没有已经接受的value，那么它可以自由决定value）。
在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即批准这个请求。<br>
<br>

回到PhPaxos，两张图可以说明以上<br>
$$<br>
![Xixia](/assets/images/8fefe554dbafa11a3961bef654759ca5_hd.png)<br>
![Xixia](/assets/images/4fa5876e919fe2b9490612a61ec2eb1d_hd.png)<br>
<br>
<br>
<br>
## 二，Zookeeper中的实现<br>
Zookeeper并没有完全实现Paxos,它实现的是zab. Zab的实现要比Paxos简单非常多
* 只有Leader才能发起Proposal.
* 其它Follow收到数据包之后无条件回复Ack.
* 超过一半的Ack，就认为该提案生效

### Leader选举
用事物ID:ZXID表示一个$\beta$
用SID表示$B_{dec}$
用Max(SID)表示MaxVote()$_{dec}$

### 修改数据
每一个修改的命令，zookeeper都会给它分配一个新的zxid
代码在
```java
//apache-zookeeper-3.5.5/zookeeper-server/src/main/java/org/apache/zookeeper/server/PrepRequestProcessor.java
public class PrepRequestProcessor extends ZooKeeperCriticalThread implements
        RequestProcessor {
   protected void pRequest(Request request) throws RequestProcessorException {
        ...
            switch (request.type) {
                ....
                case OpCode.setData:
                    ..              
                    pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
                    break;
    ...
}
```

```java
/apache-zookeeper-3.5.5/zookeeper-server/src/main/java/org/apache/zookeeper/server/quorum
    //可以看到关键的参数是zxid
     synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {        
        ....
        if (lastCommitted >= zxid) {//收到之前的消息，忽略就好了
            ....
            return;
        }
        //通过zxid获得投票
        Proposal p = outstandingProposals.get(zxid);
        ......
        //等待Ack过半，就认可修改生效
        boolean hasCommitted = tryToCommit(p, zxid, followerAddr);
    }
    
```