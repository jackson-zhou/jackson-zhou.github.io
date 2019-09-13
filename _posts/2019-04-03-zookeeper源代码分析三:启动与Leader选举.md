上文提到, Zookeeper服务端的角色分为Leader和非Leader,包括Follower,Observer. 对于所有的写请求则都会转交给Leader来处理.
那么就面临一个问题, 谁是Leader?

尝试了很久, Leader部分不适合直接分析代码,那样容易见木不见林.毕竟Leader选举是Zookeeper中最核心的技术之一.
网上其实存在着很多优秀的Leader分析文章, 我尝试着给出自己版本的分析笔记.

Zookeeper服务器当自己处于LOOKING状态时, 就会尝试进入Leader选举.
* 服务器初始化时,自己的LOOKING状态
* 服务器运行期间无法和Leader保持连接的,也会进入LOOKING状态


第二点的代码分散在各个代码中, 比较零散. 先看看第一个,当服务器初始化时,自己处于LOOKING状态,会进入Leader选举.
```Java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/QuorumPeer.java
public class QuorumPeer ...
 @Override
    public synchronized void start() {
        loadDataBase();
        cnxnFactory.start();        
        startLeaderElection();// 开始进行Leader选举
        super.start();
    }
   synchronized public void startLeaderElection() {
    	.....
        this.electionAlg = createElectionAlgorithm(electionType);//选择合适的选举算法
    }
     protected Election createElectionAlgorithm(int electionAlgorithm){
        ....
        switch (electionAlgorithm) {
        ....
        case 3:
           ...
                le = new FastLeaderElection(this, qcm);//选择FastLeaderElection算法
           ...
            break;
        default:
            assert false;
        }
        return le;
    }
    public void run() {
		....
			 while (running) {
                switch (getPeerState()) {
                case LOOKING://当自己是 LOOKING状态时, 再试选举Leader
						..
						setCurrentVote(makeLEStrategy().lookForLeader());//寻找Leader			...
				}
				 ...
			 }
		..
	}
	private ServerState state = ServerState.LOOKING;//刚启动时, 让自己处于LOOKING状态
 }
...
}
```

lookForLeader的代码其实很不好读, 很容易见木不见林.

Fast Leader Election
这部分的源码主要是在FastLeaderElection.java的lookForLeader()方法中．
然后我们再来看看FastLeaderElection的实现主干 

 当一台机器进入 Leader选举流程时,当前集群也可能会处于以下两种状态:
* 集群中已经存在着Leader
* 集群中还没不存在Leader

#### 集群中已经存在着Leader
我们先来看看集群中已经存在着 Leader的情况,换句话说我们接收到的服务器状态并不是在LOOKING状态, 那要么是FOLLOWING,要么是LEADING.

a) 如果逻辑时钟相同,将该数据保存到recvset,如果所接收服务器宣称自己是leader,那么将判断是不是有半数以上的服务器选举它,如果是则设置选举状态退出选举过程
b) 否则这是一条与当前逻辑时钟不符合的消息,那么说明在另一个选举过程中已经有了选举结果,于是将该选举结果加入到outofelection集合中,再根据outofelection来判断是否可以结束选举,如果可以也是保存逻辑时钟,设置选举状态,退出选举过程.

```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/FastLeaderElection.java
Fastleaderelection.lookforleader()代码
public Vote lookForLeader() throws InterruptedException {  
       .....  
       updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch();//收到投票时先投给自己
       .....
       sendNotifications();  //发送投票
       ...
	   switch (n.state) {  
               	...
				   case FOLLOWING:  
                   case LEADING:  
                   /*
                         * Consider all notifications from the same epoch
                         * together.
                         */
                        if(n.electionEpoch == logicalclock.get()){
                            recvset.put(n.sid, new Vote(n.leader,
                                                          n.zxid,
                                                          n.electionEpoch,
                                                          n.peerEpoch));
                           
                            if(ooePredicate(recvset, outofelection, n)) {
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(n.leader, 
                                        n.zxid, 
                                        n.electionEpoch, 
                                        n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                        /*
                         * Before joining an established ensemble, verify
                         * a majority is following the same leader.
                         */
                        outofelection.put(n.sid, new Vote(n.version,
                                                            n.leader,
                                                            n.zxid,
                                                            n.electionEpoch,
                                                            n.peerEpoch,
                                                            n.state));
           
                        if(ooePredicate(outofelection, outofelection, n)) {
                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                            }
                            Vote endVote = new Vote(n.leader,
                                                    n.zxid,
                                                    n.electionEpoch,
                                                    n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;
         ...

```

#### 集群中还没不存在Leader


```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/FastLeaderElection.java
Fastleaderelection.lookforleader()代码
public Vote lookForLeader() throws InterruptedException {  
       .....  
       updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch();//收到投票时先投给自己
       .....
       sendNotifications();  //发送投票
       ...
	   switch (n.state) {  
                //LOOKING消息，则    
                   case LOOKING:  
                    //检查下收到的这张选票是否可以胜出，依次比较选举轮数epoch，事务zxid，服务器编号server id  
                       // If notification > current, replace and send messages out  
                       if (n.electionEpoch > logicalclock) {  
                           logicalclock = n.electionEpoch;  
                           recvset.clear();  
                           if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,  
                                   getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {  
                               updateProposal(n.leader, n.zxid, n.peerEpoch);  
                           } else {  
                               updateProposal(getInitId(),  
                                       getInitLastLoggedZxid(),  
                                       getPeerEpoch());  
                           }  
                           sendNotifications();  
                       } else if (n.electionEpoch < logicalclock) {  
                           if(LOG.isDebugEnabled()){  
                               LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"  
                                       + Long.toHexString(n.electionEpoch)  
                                       + ", logicalclock=0x" + Long.toHexString(logicalclock));  
                           }  
                           break;  
                       } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,  
                               proposedLeader, proposedZxid, proposedEpoch)) {  
                        //胜出了，就把自己的投票修改为对方的，然后广播消息    
                           updateProposal(n.leader, n.zxid, n.peerEpoch);  
                           sendNotifications();  
                       }  
                       .....  
                    //添加到本机投票集合，用来做选举终结判断   
                       recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));  
                    //选举是否结束，默认算法是超过半数server同意    
                       if (termPredicate(recvset,  
                               new Vote(proposedLeader, proposedZxid,  
                                       logicalclock, proposedEpoch))) {  
  
                           // Verify if there is any change in the proposed leader  
                           while((n = recvqueue.poll(finalizeWait,  
                                   TimeUnit.MILLISECONDS)) != null){  
                               if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,  
                                       proposedLeader, proposedZxid, proposedEpoch)){  
                                   recvqueue.put(n);  
                                   break;  
                               }  
                           }  
  
                           /*  
                            * This predicate is true once we don't read any new  
                            * relevant message from the reception queue  
                            */  
                           if (n == null) {  
                            //修改状态，LEADING or FOLLOWING    
                               self.setPeerState((proposedLeader == self.getId()) ?  
                                       ServerState.LEADING: learningState());  
                            //返回最终的选票结果   
                               Vote endVote = new Vote(proposedLeader,  
                                       proposedZxid, proposedEpoch);  
                               leaveInstance(endVote);  
                               return endVote;  
                           }  
                       }  
                       break;  
                .....
}

```
参考:
[从Paxos到ZOokeeper分布式一致性原理与实践](https://blog.csdn.net/xhh198781/article/details/6619203)<br> 
[zookeeper3.3.3源码分析(二)FastLeader选举算法](http://itindex.net/detail/51448-zookeeper-fastleader-%E9%80%89%E4%B8%BE)<br>
[图解zookeeper FastLeader选举算法](https://blog.csdn.net/a040600145/article/details/53842294)<br>
[zookeeper源码浅析（二）之Leader选择](https://blog.csdn.net/zpoison/article/details/80615468) <br>
[（四）理解zookeeper设计原理及ZAB协议和Leader选举](https://blog.csdn.net/gaoshan12345678910/article/details/67638657) <br>
[【分布式】Zookeeper的Leader选举-选举过程介绍比较清晰](https://www.cnblogs.com/ASPNET2008/p/6421571.html) <br>
[理解zookeeper选举机制----zookeeper集群](https://www.jianshu.com/p/357ca7c3b2af )<br>
ZooKeeper源码解析(6)-Zab实现解析