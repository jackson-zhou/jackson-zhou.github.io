---
layout: default
title: Paxos在Zookeeper中的实现
---
# Paxos的理解
## 参考     
https://zhuanlan.zhihu.com/p/21438357  
论文地址： https://www.microsoft.com/en-us/research/uploads/prod/2016/12/The-Part-Time-Parliament.pdf  
译文：  https://www.cnblogs.com/hzmark/p/The_Part-Time_Parliament.html

主要是**2.1 Mathematical Results**
$B$: 一轮投票参与元素的集合  
$B_{bal}$这轮投票所使用的编号,表示第几轮投票
$B_{dec}$ 这轮投票的目标提议（可以为任何可描述的事物，比如一串二进制，一个图像等）  
$B_{qrm}$ 一轮 成功的投票所需要的投票者集合（可理解为多数派）  
$B_{vot}$这轮投票实际投票者集合（实际真正投票成功的投票者）  

## 投票约束   
$B1(\beta)$:$\beta$中的每一轮投票，编号必须是唯一的。  
$B2(\beta)$:$\beta$中任意两轮投票的法定人数集合，至少有一个重合的议员。可以理解为只有多数派进行了投票，本轮投票才算成功.  
$B3(\beta)$:对于$\beta$中的一轮投票编号为$B_{bal}$的投票， 假如多数派中存在任意一名成员投过比$B_{bal}$编号小的票$B'$， 那么$B_{dec}=B'_{dec}$ 

可以用数学符号来描述以上约束：

$$
\begin{aligned}
& B1(\beta) \triangleq \forall B, B' \in \beta :  (B \neq B')\Rarr (B_{bal}\neq B'_{bal}) \\
& B2(\beta) \triangleq \forall B, B' \in \beta :  B_{qrm} \cap B'_{qrm}=\empty \\
& B3(\beta) \triangleq∀B ∈ \beta : (MaxVote(B_{bal}, B_{qrm}, \beta)_{bal} = -\infin) \Rarr \\ 
 & \mskip{4cm} (B_{dec} = MaxVote(B_{bal}, B_{qrm}, \beta)dec)
\end{aligned}
$$
![Xixia](/assets/images/4fa5876e919fe2b9490612a61ec2eb1d_hd.png)
![Xixia](/assets/images/4fa5876e919fe2b9490612a61ec2eb1d_hd.png)