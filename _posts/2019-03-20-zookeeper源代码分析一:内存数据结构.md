---
layout: default
categories: [源码分析,分布式,java]
---
其实阅读代码就像是走迷宫,从main函数开始,一步一步地去推导,去试验,去小结软件的每一个功能.
但反过来,如果借鉴走迷宫的方法,直接从终点倒推回起点,就会变得更简单.
例如从zookeeper开始. zookeeper是一个分布式应用程序协调服务. 反过来说,它将分布式的一系列问题抽象成了一个内存树以及对这个内存树的操作,外加一个通知机制.这就构成了zookeeper.

我们可以从这个角度入手, 去看看zookeeper的源代码, 看看它是怎么实现的.本次阅读的代码基于最新稳定版3.4.12:
[官网](https://zookeeper.apache.org/doc/current/zookeeperOver.html)在很靠前的位置介绍了这棵树:
<img src='/static/image/code_analyze/19c0614ff4b52173a466c936034e5458.png' />

这棵树的代码在:
```java
  //zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/DataTree.java
public class DataTree {
  ........
  /**
     * This hashtable provides a fast lookup to the datanodes. The tree is the
     * source of truth and is where all the locking occurs
     */
  private final ConcurrentHashMap<String, DataNode> nodes =
        new ConcurrentHashMap<String, DataNode>();
  ....
   /**
     * This hashtable lists the paths of the ephemeral nodes of a session.
     */
  private final Map<Long, HashSet<String>> ephemerals =
        new ConcurrentHashMap<Long, HashSet<String>>();
  ...
  public ProcessTxnResult processTxn(TxnHeader header, Record txn)
    {
      ...
            switch (header.getType()) {
                case OpCode.create:
                    ....
                    break;
                case OpCode.delete:
                    ....
                    break;
                case OpCode.setData:
                    ....
                    break;
                case OpCode.setACL:
                    ....
                    break;
                case OpCode.closeSession:
                    ....
                    break;
                case OpCode.check:
                    ....
                    break;
                case OpCode.multi:
                    ....
                    break;
        ....
    }
 }
```
从这个代码可以看到几个重要的信息:
* zookeeper提供的写命令对应的写操作都在这个类里.
* zookeeper的内存数据其实是一个平面的hash表, key是路径名, value是一个DataNode. hash表不具有层级结构----从setData函数可以验证这一点.
* zookeeper的节点分为永久节点和临时节点, 临时节点跟Session相关. zookeeper的处理方式是在**ephemerals**中对于每个Session,都用sessionid为key,记录了相关路径名称. 在Sesssion失效后,遍历ephemerals删除相应的节点. 具体代码参见:killSession
* 对于内存数据的操作,会触发响应的watcher回调. 参见分散的处理函数里的最后,会有:dataWatches.triggerWatch...
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/WatchManager.java
public class WatchManager {
	public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
	......
		//可以看到zookeeper每次触发一个事件,都会将它从watchTable里拿掉
		watchers = watchTable.remove(path);
	.....
		for (Watcher w : watchers) {
			.....
				w.process(e);
		}
	....
	}
}
```

从这个triggerWatch来看, 所有的路径watch都是一次性的. 每次一触发都会从watchTable中拿掉.
当客户端Session超时引起closeSession, 从而触发临时节点被删除时,也会触发这个trigger.不过这时客户端已经掉线从而收不到这个通知.
网上看到的很多踩坑zookeeper的案例也正是由于这个特性引起的!

甚至还有为了躲避这个特性的代码而成为apache的顶级项目:[curator](http://curator.apache.org/), 真是令人无语.....

在我的角度, zookeeper这么设计是有它的道理的. 之前的文章提过分布式是一个恶劣的服务运行环境,这个环境里充满了各种超时,也缺乏全局时钟.因此作为分布式协调系统,它更关心的是给它的客户端一个可靠的结果. 不需要把每次数据的变化都通知给客户端.
所以zookeeper的设计里,<font color="#FF550">通知不带数据! 通知只是一次性的!</font>  zk客户端收到变化通知以后,需要根据不同的业务场景自行决定是否需要拉取最新数据(sync),拉取的时机, 以及是否需要注册下一次的通知.

在某些场景下,一收到变化通知就立刻从zookeeper拉取最新数据并不是个好主意.
参见:[Zookeeper，你可把我坑惨了！](https://www.jianshu.com/p/81601bb6914b), 在这个案例中作者用zk存储路由规则. 路由一变,一通知,近两千台机器同时从zookeeper服务器上获取数据.于是流量打死了物理机，打死了专线.从而又引起新一轮的路由规则变化.死循环了.

同样的, 收到通知变化后立刻再统一节点下注册watch事件也并不是个好主意, 它违背了zookeeper设计的本意. 更不要妄想去接受zookeeper数据节点的每一次变化数据. zk只是一个协调服务不是一个消息队列, 如果需要收到数据的每次变化应该改用MQ一类的服务.


继续往外阅读代码, 这个DataTree定义在:
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/ZKDatabase.java
public class ZKDatabase {
	...
	protected DataTree dataTree;//在这里
	...
	 public DataTree getDataTree() {
        return this.dataTree;
    }
	...
	public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
        return dataTree.processTxn(hdr, txn);
    }
	..
}
```
```java
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/ZooKeeperServer.java
public class ZooKeeperServer implements SessionExpirer, ServerStats.Provider {
	...
	private ZKDatabase zkDb;//ZKDatabase在这里
	...
	/**
     * get the zookeeper database for this server
     * @return the zookeeper database for this server
     */
    public ZKDatabase getZKDatabase() {
        return this.zkDb;
    }
	...
	public ProcessTxnResult processTxn(TxnHeader hdr, Record txn) {
		...
		rc = getZKDatabase().processTxn(hdr, txn);
		....
	}
	...
}
```

再往外看, Zookeeper Server的所有操作都来自zk客户端,通过网络消息传递.
于是可以看到消息处理函数
```java 
//zookeeper-3.4.12/src/java/main/org/apache/zookeeper/server/FinalRequestProcessor.java
public class FinalRequestProcessor implements RequestProcessor {
	....
	public void processRequest(Request request) {
		...
		rc = zks.processTxn(hdr, txn)
		..
	}
}
```
再往外就涉及到了zookeeper的请求处理链. 敬请期待下篇: zookeeper源代码分析之:请求处理链