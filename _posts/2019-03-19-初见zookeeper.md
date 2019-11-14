---
layout: default
categories: [源码分析，java]
---   
作为一个资深C++程序员, 第一次用java写zookeeper客户端程序, 我完全被震撼到了.
用Eclipse创建Maven工程,规范性地填写你的代码组织(groupId),还有工程名称(artifactId). 很容易地创建了一个helloworld的java工程. 从maven仓库(https://mvnrepository.com/) 查找zookeeper的索引信息往pom.xml文件里一贴. eclipse一阵忙乎,所有的依赖环境就自动解决了.
```xml 
<!-- https://mvnrepository.com/artifact/org.apache.zookeeper/zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.13</version>
    <type>pom</type>
</dependency>
```

可以放心大胆地在代码里写上
```java
ZooKeeper zk = new ZooKeeper("192.168.0.100:2181", 3000,null );
byte[]  td = zk.getData("/test", monitor, null);
System.out.println("test=" + new String (td));
```
代码就能运行了, 就能正确地得到zookeeper集群里的测试数据了. 震惊得我半天说不出话来. 这java生态强悍得没话说, 也能理解为什么C/C++这些年来颓废得这么厉害.
最近的C++20即将发布,也没见在分布式,生态环境这方面有太多的努力, 不免觉得有些悲哀.

很顺利地,几乎没有障碍地写完zk的helloworld
```java
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import java.util.List;

public class zktest {
    public static void main(String[] args) {
		//只是个hello world版本,没有采用生硬的面向对象:先new zktest, 再执行zktest.run(0)
        try {
			//惊讶, 直接在构造函数里完成连接操作,够狠. 这个在C++里通常是禁止写法.
            ZooKeeper zk = new ZooKeeper("192.168.0.100:2181", 3000,null );
            DataMonitor monitor = new DataMonitor();
            monitor.Set(zk);//没什么意思,只是为了在monitor里重新设置watcher

            try {
                /*
				//惊讶吧, 完全把复杂的网络通讯,协议处理包装起来了.直接一个getChildren了事.
                List<String> listChild = zk.getChildren("/test", monitor);
                for (String str : listChild) {
                    System.out.println("->" + str);
                }
                */
				
				//惊讶吧, 在eclipse的提示下,不用看文档都知道除了这个同步读取方式外,还有异步读取方式.
                byte[]  td = zk.getData("/test", monitor, null);
                System.out.println("test=" + new String (td));
                //只是为了让main函数不退出, 懒得去查java的表达方式,一个死循环了事.
				int i = 0;
                while (true) {
                    System.out.println("wating..." + String.valueOf(i++));
                    Thread.sleep(50000000);
                    //zktest.class.wait();
                }
            } catch (KeeperException ke) {//java也太狠了,动不动就会来个异常.是不是有滥用的嫌疑?
                System.out.println("KeeperException:" + ke.toString());

            } catch (InterruptedException ie) {
                System.out.println("InterruptedException:" + ie.toString());
            }
            System.out.println("Hello, This is zktest");
        } catch (java.io.IOException e) {
            System.out.print("Error:");
            System.out.println(e);
        }
    }
};

//一个回调类
class DataMonitor implements Watcher {
    ZooKeeper zk;
    public void Set(ZooKeeper zk) {
        this.zk = zk;
    }
    public void process(WatchedEvent event) {
        String path = event.getPath();
        System.out.println("path:" + path + ", has changed");
        System.out.println(event);
        if (path != null) {
            // Something has changed on the node, let's find out
            try {//太狠了,一定要用异常,而且还一定要处理异常.
                Stat stat = new Stat();
                byte[]  dd = zk.getData("/test", this, stat);//重新注册watcher
                System.out.println("process test=" + new String (dd));
            } catch (KeeperException ke) {
                System.out.println("DataMonitor KeeperException:" + ke.toString());

            } catch (InterruptedException ie) {
                System.out.println("DataMonitor InterruptedException:" + ie.toString());
            }
        }
    }
}
```
扣掉不得不写的异常处理,强制性的回调类包装. zookeeper的使用方式简单的令人发指.
通过文档也可知,zookeeper把复杂的分布式问题抽象成一颗内存树,6个API表达完. 震撼!!!
6个API分别是
* create
* delete
* exists
* getData
* setData
* getChildren

还有一个用来获取最新数据的
* sync

通过简单的6个API, Zookeeper将恶劣的分布式环境下编程抽象成对一颗内存数的访问.
异构性?用SDK方式屏蔽了.不用关心编程语言,通讯协议,只需要找到对应语言的SDK即可.
缺乏全局时钟? zk提供了顺序一致性. 尤其是统一客户端发起的事务请求,不管客户端是否因网络问题进行了重试, zk最终会严格按照其发起顺序应用到服务器中去(setData带了版本号). 
zk还提供了单一视图, 不管你连接到了zk机器里的那个服务器, 你看到的服务端数据模型都是一致的.

总之: zookeeper的设计目标就是简单, 可复制, 有序以及快速.