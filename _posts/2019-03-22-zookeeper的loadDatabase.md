---
layout: default
categories: [源码分析,分布式,java]
---
<script src="/assets/mermaid.min.js"></script>

<div class="mermaid">
    graph TB;
	loadDatabase-->A(获取最多100个snapshot.xxx文件)
	A-->B(找到第一个有效的snapshot文件,并反序列化到内存的DataTree)
	B-->C(读取snapShot文件,得到lastProcessedZxid)
	C-->D(lastProcessedZxid以前的数据都在snapshot里了, 更新的数据在txnLog)
	D-->E(从lastProcessedZxid+1开始找txnLog)
	E-->F(对每个transaction,在内存replay,同事发送给learners)
	F-->G(得到当前最新的zxid,初始化完毕)
</div>