# Paxos协议

Paxos用于实现分布式系统的一致性,分为两个阶段。

## 基本概念
Proposal Value：提议的值（用val表示）  
Proposal Number：提议编号（可以是时间戳，用ID表示）  
Proposal：提议 = 提议编号 + 提议的值（ID，Val）  
Proposer：提议发起者  
Acceptor：提议接受者  

## 第一阶段（获取访问权）
1. proposer向所有的acceptor申请访问权，即调用prepare(ID)。  
2. acceptor保存了maxId，和(cID,cVal)，初始化均为null。maxId是持有访问权的ID，小于maxID的ID的访问权失效，在第一阶段修改；（cID，cVal）是已接受的提议，在第二阶段修改。acceptor收到提议ID后，有2种情况：  
    1. maxId为空（第一次申请访问权）或ID>maxId，则将maxId更新为ID,并返回(ok,cID,cVal),其中cVal可能有值也可能为null，颁发访问权。
    2. ID<=maxId,返回(error)，不颁发访问权。
3. 若proposer获取到半数以上访问权，则进入第二阶段。否则生成新的提议ID，重新执行第一阶段。

## 第二阶段（修改值）
1. proposer向获取到访问权的acceptor进行提议，即调用propose(ID,Val)，由于能够进入第二阶段，说明ID已经被半数以上acceptor所接受，关于Val的取值，分2种情况：
    1. 如果存在非空cVal，则选择cID最大的一个对应的cVal，作为Val。
    2. 如果都是空的cVal，则用自己原本准备提交的value作为Val。
2. acceptor收到提议后，首先判断ID是否与保存的maxId相等（判断ID是否有访问权），如果不是，则返回(error)；如果是，则更新cID=ID,cVal=Val。
3. proposer如果收到半数以上成功，则返回(ok,Val),否则返回(error)（被新ID抢占或者acceptor故障）。

## 其它
### 活锁
假设只有节点1和节点2，首先节点1第一阶段获得所有节点访问权，接着节点2第一阶段抢占所有节点访问权，此时节点1执行第二阶段会失败，因为访问权被节点2抢占，于是节点1重新执行第一阶段抢占所有节点的访问权，又导致节点2无法执行第二阶段。如此往复，形成活锁。  
因为只要某个节点抢占时间晚一点，锁就会自动解开，所以不是死锁。

### proposer的半数以上访问权
如果某个决议（假设是将s设置成2）被确定，则一半以上的节点中s均为2，令这些节点构成集合S1。后续某个新的提议若想被执行，也需要获得半数以上的访问权，新提议获得访问权的节点中必然有一个是集合S1中的节点，因此新的提议知道s的值需要被设置成2。

### proposer 如何确定
工程实现上一般一个实例同时承担Proposer和Acceptor两个角色

