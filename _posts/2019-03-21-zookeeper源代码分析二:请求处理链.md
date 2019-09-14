本文打算分析一下zookeeper读写消息的请求处理链, 附带分析一下zookeeper服务器的几个角色(Leader/Follower/Observer).

让我们从[官网](https://zookeeper.apache.org/doc/current/zookeeperOver.html)上的一张图,一段描述开始:
![](http://jacksonzhou.top/wp-content/uploads/2019/03/5ad5bd279b88825507f70087b321f33a.png)

<p></p><p></p><p></p>
从官网的描述可知, 不管你连接zookeeper集群的哪台服务器, 所有的写请求都会被转发到Leader服务器,并且在一个集群中Leader只有一个.
这就是我们分析请求链很好的一个起点. <font color="#FF1122">看Zk的写流程消息处理链只需要看Leader的就够了</font>.  

# Leader的消息处理链

```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/LeaderZooKeeperServer.java
public class LeaderZooKeeperServer extends QuorumZooKeeperServer {
	...
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(
                finalProcessor, getLeader().toBeApplied);
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false,
                getZooKeeperServerListener());
        commitProcessor.start();
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        proposalProcessor.initialize();
        firstProcessor = new PrepRequestProcessor(this, proposalProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
	...
}
```

从代码中可以看到Leader的消息处理链分别是
<script src="/static/mermaid.min.js"></script>

<div class="mermaid">
    graph LR;
        PrepRequestProcessor-->ProposalRequestProcessor;
        ProposalRequestProcessor-->CommitProcessor;
        CommitProcessor-->ToBeAppliedRequestProcessor;
        ToBeAppliedRequestProcessor-->FinalRequestProcessor;
</div>    

再细看一下
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/ProposalRequestProcessor.java
public class ProposalRequestProcessor implements RequestProcessor {
  ...
    public ProposalRequestProcessor(LeaderZooKeeperServer zks,
            RequestProcessor nextProcessor) {
        this.zks = zks;
        this.nextProcessor = nextProcessor;
        AckRequestProcessor ackProcessor = new AckRequestProcessor(zks.getLeader());
        syncProcessor = new SyncRequestProcessor(zks, ackProcessor);
    }
    ...
	    public void processRequest(Request request) throws RequestProcessorException {
        // LOG.warn("Ack>>> cxid = " + request.cxid + " type = " +
        // request.type + " id = " + request.sessionId);
        // request.addRQRec(">prop");
                
        
        /* In the following IF-THEN-ELSE block, we process syncs on the leader. 
         * If the sync is coming from a follower, then the follower
         * handler adds it to syncHandler. Otherwise, if it is a client of
         * the leader that issued the sync command, then syncHandler won't 
         * contain the handler. In this case, we add it to syncHandler, and 
         * call processRequest on the next processor.
         */
        
        if(request instanceof LearnerSyncRequest){
            zks.getLeader().processSync((LearnerSyncRequest)request);
        } else {//这个是Leader
			//先交给CommitProcessor处理, 注意:Commit线程(run函数)会等待投票通过.
            nextProcessor.processRequest(request);
            if (request.hdr != null) {
                // We need to sync and get consensus on any transactions
                try {
					//发起一个投票, CommitProcess会等待这个投票通过
                    zks.getLeader().propose(request);
                } catch (XidRolloverException e) {
                    throw new RequestProcessorException(e.getMessage(), e);
                }
				//调用Sync
                syncProcessor.processRequest(request);
            }
        }
    }
```

也就是Leader真实的消息处理链其实是:  

<div class="mermaid">
    graph LR;
        PrepRequestProcessor-->1,ProposalRequestProcessor
        1,ProposalRequestProcessor-->3,CommitProcessor
        3,CommitProcessor-->ToBeAppliedRequestProcessor
        ToBeAppliedRequestProcessor-->4,FinalRequestProcessor
        1,ProposalRequestProcessor-->SyncRequestProcessor
        SyncRequestProcessor-->2,AckRequestProcessor
</div>


到了FinalRequestProcessor时,就直接操作内存数据库了.
但是请求消息在到达Final之前, 会先经过CommitProcessor, 而CommitProcessor里则会等待一个投票结果. 投票过程在ProposalRequestProcessor里发起.

所以让我们先看看CommitProcessor里等待的内容,在看看它等到的投票什么时候达成. 这段代码比较绕啊, 之后会开篇文章重新讲解
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/CommitProcessor.java
public class CommitProcessor extends ZooKeeperCriticalThread implements RequestProcessor {
	...
    public void run() {
	...
		if ((queuedRequests.size() == 0 || nextPending != null)
			&& committedRequests.size() == 0) {//在这里等待commitRequest
			wait();
			continue;
		}
	...
	}
...
}
```
```java
/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.apache.zookeeper.server.quorum;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import org.apache.zookeeper.server.Request;
import org.apache.zookeeper.server.RequestProcessor;


/**
 * This is a very simple RequestProcessor that simply forwards a request from a
 * previous stage to the leader as an ACK.
 */
class AckRequestProcessor implements RequestProcessor {
    ...
    public void processRequest(Request request) {
    ...
            leader.processAck(self.getId(), request.zxid, null);
	...
	}
...
}

```
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/Leader.java
public class Leader {
	...
	synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {
	 	if (self.getQuorumVerifier().containsQuorum(p.ackSet)){//如果满足投票条件, 则进行commit. 至于满足什么投票条件?之后再分析             
            .....
            zk.commitProcessor.commit(p.request);
            ...
        }
	...
	}
	...
}
```

再往前一点的ProposalRequestProcessor里有个细节,很容易被遗漏的. 再看一眼
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/ProposalRequestProcessor.java
public class ProposalRequestProcessor implements RequestProcessor {
  ...
    public void processRequest(Request request) throws RequestProcessorException {
        ......
        if(request instanceof LearnerSyncRequest){
            ......
        } else {//这个是Leader
			zks.getLeader().propose(request);//发起一轮投票
			....
            syncProcessor.processRequest(request);//间接调用AckRequestProcessor, 投了自己一票.
         
        }
    }
```
SyncRequestProcessor的代码读注释就够了, 比看代码简介.
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/SyncRequestProcessor.java
/**
 * This RequestProcessor logs requests to disk. It batches the requests to do
 * the io efficiently. The request is not passed to the next RequestProcessor
 * until its log has been synced to disk.
 *
 * SyncRequestProcessor is used in 3 different cases
 * 1. Leader - Sync request to disk and forward it to AckRequestProcessor which
 *             send ack back to itself.
 * 2. Follower - Sync request to disk and forward request to
 *             SendAckRequestProcessor which send the packets to leader.
 *             SendAckRequestProcessor is flushable which allow us to force
 *             push packets to leader.
 * 3. Observer - Sync committed request to disk (received as INFORM packet).
 *             It never send ack back to the leader, so the nextProcessor will
 *             be null. This change the semantic of txnlog on the observer
 *             since it only contains committed txns.
 */

```

小结一下:
Leader的流程就是:
1,收到消息
2, 向所有的成员发起投票
3, 投自己一票.
4, 等待投票过半
5, 修改内存数据.


# Follower的消息处理链
看完了Leader, 往外再看看 Follower
预期中的, 当Follower收到写请求时会转发给Leader
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/FollowerRequestProcessor.java
public class FollowerRequestProcessor ...{
	...
	public void run() {
		...
		switch (request.type) {
		...
		case OpCode.create:
		case OpCode.delete:
		case OpCode.setData:
		case OpCode.setACL:
		case OpCode.createSession:
		case OpCode.closeSession:
		case OpCode.multi:
			zks.getFollower().request(request);//转发给leader
			break;
		}
	...
}
```

在FollowerZookeeperServer.java中可以看到完整的请求处理链
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/FollowerRequestProcessor.java
public class FollowerZooKeeperServer extends LearnerZooKeeperServer {
	....
	 @Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);  //第三个请求链节点.
        commitProcessor = new CommitProcessor(finalProcessor,  //第二个请求链节点.
                Long.toString(getServerId()), true,
                getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new FollowerRequestProcessor(this, commitProcessor);//第一个请求链节点.
        ((FollowerRequestProcessor) firstProcessor).start();
        syncProcessor = new SyncRequestProcessor(this,
                new SendAckRequestProcessor((Learner)getFollower()));//独立的请求链点
        syncProcessor.start();
    }
}
```
在Follower.java中可以看到
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/quorum/Follower.java
...
public class Follower extends Learner{
	...
	final FollowerZooKeeperServer fzk;
	...
	 protected void processPacket(QuorumPacket qp) throws IOException{
        switch (qp.getType()) {
        ...
        case Leader.PROPOSAL:            
           ....
           fzk.logRequest(hdr, txn);//调用syncProcessor,ackProcessor
```


也就是对于Leader发起的投票PROPOSAL, 它会立即返回ack.

参考资料:
[参考深入浅出Zookeeper之七分布式CREATE事务处理](https://iwinit.iteye.com/blog/1777109)<br>
[一篇文章看懂ZooKeeper内部原理](https://blog.csdn.net/taurus_7c/article/details/81143830)<br>
[ZooKeeper源码分析：Quorum请求的整个流程(流程交互图)](https://blog.csdn.net/jeff_fangji/article/details/42988439)<br>
[zookeeper源码分析之五服务端(集群leader)处理请求流程](https://www.cnblogs.com/davidwang456/p/5004599.html)<br>
[zookeeper源码浅析（一）交互图](https://blog.csdn.net/a040600145/article/details/53842280)<br>




