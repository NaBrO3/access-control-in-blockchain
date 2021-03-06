## 概述

区块链上的个人数据（本文主要讨论文件类型的数据，不包括交易，账户等）。管理包含两部分内容：

- 访问控制
- 数据保护

应用：


- 数据共享
- 云盘
- 医疗系统
- IOT
- 版权


目标：

- 灵活授权撤销
- 细粒度访问控制
- 避免单点攻击/失败
- 避免集中控制，随意篡改；分布式控制，透明，无法做恶
- 降低数据泄露风险



## 方案

~~### 分布式存储系统（ipfs）
无法灵活控制授权撤销~~

### 链上管理合约+链下集中式访问控制



数据的控制访问规则以及授权撤销均通过链上来执行。

requester请求数据时，data server会查询链上数据以确定其是否有权访问。

数据存在链下。此时，数据的哈希会上链或者带有签名，以保证数据的完整和真实性。

缺点：依赖中心化服务器，需要很强的信任假设。

改进：可以将中心化服务器替换为TEE。


~~### TEE
可以穿插于各个方案中~~


### 数据隔离（物理）+ACL


需要链底层支持，不适用于已有公链

对参与节点需要很强的信任假设，在公链上比较难以实施，更适用于联盟链

目前，大部分联盟链都是采用这种方式。


**hyperledger fabric**

https://hyperledger-fabric.readthedocs.io/zh_CN/release-1.4/glossary.html#anchor-peer

raft:  https://www.geekmeta.com/article/1237586.html
问题1：排序服务使用RAFT共识，RAFT的前提是节点诚实可信，可若结点都诚实可信了，构建区块链意义何在
问题2：同一区块中若有多笔关联交易，则第一笔成功，其余均失败
（参考file:///C:/Users/syy/Documents/temp/NewSpiralWhitePaper.pdf）

有Channel的概念，通道中的数据与网络的其他部分(包括其他通道)完全隔离，一个个通道类似于一个个子链。

私有数据集合(Pirvate Data Collection)， 解决的是如何在一个通道的部分机构之间共享数据，每一个集合都会有policy（包括分发策略和访问策略ACL），policy可更新，私有数据所存节点会查询请求节点是否在policy的允许范围。私有数据哈希会保存在channel中，其他节点也可看见。

私有数据的数据库和通道中的账本是分开存储的，每个私有数据集都对应一个独立的存储。

![](./img/2.png)



本质还是隐私数据授权节点隔离存放，访问时需要检查权限，依旧依赖于可信的中心化节点。

**corda**


Corda 网络使用点对点的消息传输而不是全局广播。也就是说协调一个关于账本的更新需要网络上的参与者明确的指定需要发送什么信息，发送给谁，按照什么顺序发送。

在 Corda 中，使用公证池可以防止双花问题。 公证池是一组节点 - 通常是一组操作拜占庭容错一致性算法（byzantine fault-tolerant consensus algorithm）的相互不信任的节点。


![](./img/4.png)

Corda消除了网络上所有参与者需要了解每一笔交易的需求，因为只有那些参与其中的人才会对它们感兴趣。只有当所有相关方都接受了所提交的交易的输入和输出是正确时，才会提交交易。如果有任何人不同意，那这笔交易就不会发生。然后，每笔交易都要求公证人（notary）签字才有效。notary的数据是“局部”的，每个notary只掌握经过自己公证的交易，这也是Corda“信息部分可见”特性的根本保障。

![](./img/3.png)


没有区块链的“类区块链”系统。采用了UTXO模型的系统，其交易之间实际上就有一个“链式结构”：一个交易的输出，成为另一个交易的输入，交易与交易之间就通过这个方式被串起来了。这样的链式结构，实际上是一个有向无环图（DAG）。同时，由于Corda系统的数据不是全局的，所以这样的“链”在系统中会存在多个，相互之间没有连接。因此，Corda系统中虽然没有“区块链”，但是仍然有“交易链”。

Corda中交易的类型有两种：普通交易和Notary变更交易。Corda网络中的状态具有一个Notary属性，这个Notary是当初产生这个状态的交易验证时所使用的Notary。同时，为了实现上述验证过程，Corda网络也要求一个交易的输入项必须指向同一个Notary，也就是这个交易要提交验证的Notary。如果有些状态不是指向这个Notary，则首先要对这些状态执行Notary变更交易来实现转移。此时，就需要不同Notary间的一个全局共识。全局共识就是notary变更必须要所有notary节点都达成共识，保证一个对象变更两次notary的行为（会造成双花）受到阻止。



事实上，corda整个网络被notary划分成若干相互隔绝的子网，但是，可通过变更notary允许跨子网行为的发生。





### 基于数据加密的访问控制

本质上是将数据的访问控制转移到加密密钥的获取，在这种方案下，数据可以加密后直接存于链上或者分布式存储系统IPFS等，可公开访问，但是无法解密。

优点：适用于数据链上类场景

缺点：一旦拥有权限（密钥）便难以撤销。


#### 传统加密

为每个文件新增一个对称加密密钥（也可以好几个文件对应一个密钥，甚至也可以为单个文件内每一个条目分配相应的密钥，根据访问粒度而定），使用接收者公钥对该密钥加密并发送即为授权访问。

缺：

- 随着文件数量增加，holder需要管理的对称加密密钥数量增多，授权工作量也会增加。如果需要更细粒度，密钥数量会成倍增加
- 数据owner需要在线才能完成授权的动作。



#### ABE 属性基加密




在密文里面内嵌属性访问控制逻辑，需要有一个可信的密钥分发中心，在用户满足属性条件的情况下，密钥分发中心使用主密钥和用户公钥和属性生成文件的解密密钥给该用户。

优点：细粒度，群体/批量控制

存在问题：在传统的ABE方案中，密钥分发中心是一个可信的中心化组织。这里怎么来实现密钥分发中心是一个问题，中心化存在信任问题，又不能用合约来实现，因为这样主密钥就暴漏了。可以让数据拥有者自己运行一个线下节点。每来一个用户就要自己去生成一个密钥发到链上。这样的话，如果要做到实时响应，数据owner需要在线才能完成授权的动作。但是，如此一来，该方案相比于传统方案便没有优势，反而复杂化了。可以考虑使用多授权中心的方案。

#### 代理重加密



![](./img/1.png)

owner每增加一篇文档，传统加密需要为文件新增一把密钥，且为每个接受者使用接收者公钥对文件密钥加密并发送，用代理重加密则可以省去这些操作。



优点：

- owner无需实时在线，自己运行节点。（但是，新用户初次请求数据，owner也需要在线，为其生成rekey）
- 无需为每一个文件分配并管理众多密钥

存在问题：需要一个半可信的proxy，proxy可以直接用合约来做，但是可能会增加链上计算量，可以只代理重加密加密数据的对称密钥，这样计算量不会很大。但是，用合约来做还有一个问题，rekey会暴露，别人就可以部署一个一模一样的合约，然后就可以随意生成文件密钥。所以，也是不可行的。

只能放到链下来做，那么就需要半可信中心化节点。可以用TEE/多方安全计算MPC来做，但是MPC实现相当复杂。




#### 秘密共享


秘密共享方案， 阈值秘密共享是一种通过将秘密分解给一组用户来保护秘密的方法，只有符合条件的用户子集才能通过组合他们的共享来重构秘密。



系统内有一种特殊的节点，access node，专门用来控制密钥的分发，但其又不拥有完整的密钥，数据owner根据场景选择一定数量的access node来分发影子秘密，这些节点会向满足条件的用户暴露影子秘密，条件会从链上的访问控制列表来读取，访问控制列表可以是ACL，也可以是基于属性的条件。条件由数据owner来定义。










![](./img/6.png)



优点：

- 用户不需要自己实时在线来响应请求
- 需要部分但又不完全信任access node。把信任转移至授权节点，对授权节点的信任度取决于秘密合成规则的设置，且每个授权节点是没有秘密的完整数据的。
- 可细粒度控制
- 灵活授权撤销

问题：

1.怎么做能让策略变化的时候能让以前的密钥解不开密文？

策略变化，更新密文。以前的密钥解不开当前状态的密文，但是这也可能造成以前和现在均有权限的用户需要重新申请密钥，而且密文虽然更新了，但是如果是数据直接存在链上的情况，之前的密文也是可以恢复的，只是稍微麻烦一些。

2.怎么保证不同的requestor难以勾结作恶？？怎么授权固定时间段可访问？？

shares由contentid，requesterid,timestamp，影子秘密共同生成（可以由数据owner直接生成，缺点数据owner需要实时在线）。requesterid保证不同requester的secret片段不能组合计算出secret，timestamp保证在授权时限内获取。怎么能让access node只有影子的情况下，根据contentid，requesterid,timestamp，来重构shares给用户。（难以实现，目前没看到方案）


#### 盲解密

系统中有三种KEY，

- data key：数据的加密密钥
- access key：访问策略所对应的key，被authority node所持有
- control key：访问策略所对应的key，被manage node所持有

access key+control key可以恢复data key。

access key和control key均可采取秘密共享的方案（可根据场景而定），requester获取access key的片段之后，组合成为完整的access key。经过盲化，发送给manage node，manage node检查访问策略，根据链上信息以及access node，结合自己的control key，解密出盲化后的的data key发送给requester，requester去盲化之后即可得到data key，进而解密数据。control key是不会暴露给用户的。

使用盲解密的意义在于，用户不能自己解密，必须要通过manage node来解密，但是manage node又看不到data key。

该方案不会把秘密的真正片段暴漏给所选节点（authority node 或者 manage node），进一步降低了对于node的信任度。除非authority node 和 manage node联合作恶。

相比于上面的秘密共享方案，该方案优点：



- 策略变化时能让以前的密钥解不开密文（当然，原来已经解密出明文的这种情况是没有办法解决的）
- 可以实现固定时间段解密
- 可限次解密（不过限次的意义似乎并不大）



## 总结

对于联盟链来说，物理上的数据隔离采纳较多，能达到的效果也更好，缺点是对节点所需的信任也更强。

对于公链来说，数据隔离方案实施困难，加密方案更为适用，其中盲解密方案相对来说效果最好，但也仍然存在很多待解决的问题，过程比较复杂，访问速度相对更慢，访问权限难以收回，难以达到传统中心化服务器访问控制的效果。





