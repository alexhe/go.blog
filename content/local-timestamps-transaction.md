刚进公司的时候，听同事说有个喷子经常黑我们，网上各种活跃，但技术实在low，所以大家都不屑于跟他辩，只是以逗他为乐：人被狗咬了，不可能咬回去，是吧？出于好奇心我去关注了一下，要看看才能做评价。这年头，技术水平对不上知名度的的实在太多了，毕竟在网上吹得自己秒天秒地不需要任何成本。看了他写的一篇基于局部时间戳的分布式事务模型，我就呵呵哒。所有就这个标题写一写，基于局部的时间戳来实现分布式事务，到底能不能呢？

本文准备分两篇来讲，上篇先引出问题，下篇标题还用这个，只不过内容是讲混合逻辑时钟。

实现事务的核心，是处理好事务冲突。处理冲突可以通过锁或者多版本。基于MVCC实现事务会要求版本之间能判断先后顺序，因为有先后才知道操作的数据应该用哪一个版本，而先后顺序就涉及到跟时间有关。在[分布式事务实现](distributed-transaction.md)中会遇到一个难题，就是时钟问题。时钟问题在分布式下面难搞，是因为不同机器之间的本地时钟是无法保证一致的。解决方式要么采用一个中心化的时钟服务，要么像Google能搞出高大上的TrueTime。本地时钟是没法用的。

假设一个机器在读A，另一个机器在写A，读操作是应该读到写之前的值，还是写之后的值？是要根据两个操作谁先谁后。即使有了多版本让读写不冲突，写操作会造成数据有多个版本，读也是要根据读发生在写之前，还是发生在写之后，来决定读哪一个版本。而这个先后关系的判断，当然不能够简单地用各个机器自己的本地时间。

那么用[逻辑时钟](logical-clock.md)行不行？逻辑时钟也算是局部时间戳，没有要求一个中心化的或者全局的时钟，而且逻辑时钟似乎是可以检测到事务之间有冲突的。

逻辑时钟是用节点之间的消息通信来确定事件之间的先后顺序，严格地说，应该是因果关系而不叫先后顺序。这个概念叫做happens before。在同一台机器上面发生的事件A和B，因为A发生在B之前，所以A happens before B。在两台机器之间，一台机器给另一台机器发了条消息，发出消息记为事件A，收到消息记为事件B，那么A happens before B。逻辑时钟能够保证：

    如果事件A happens 事件B，那么事件A的逻辑时钟一定小于事件B。
    
初看似乎没什么作用，因为得到了逻辑时钟不能用于比较事件发生的先后顺序：即使我知道事件A发生的时间是T1，事件B发生的时间是T2，明明满足T1 < T2，但是却不能就此判断 A happens before B。

其实有用，它可以检测冲突，把上面那句话反过来读：

    如果事件A的逻辑时钟大于事件B，那么事件A一定不是 happens before 事件B的。
    
也就是说我知道了事件A发生的时间是T1，事件B发生的时间是T2，并且有T2 > T1，那么B肯定不是 happens before A的。而对于判断冲突，只要能检测到不是happens before，就够了。也就是说，满足happens before，能保证一定安全。不能满足happens before，可能不安全，也可能安全。冒着宁可妄杀一千，不能漏过一个，我们只要发现不能满足happens before，就当冲突处理。

拿一个例子，看看逻辑时钟是怎么用的。假设一个数据，它是有多个版本的，当前已提交的版本，值是5，时间是T0，还有一个未提交的版本，值是10，时间是T1。存在多个版本说明有操作**正在**写这个数据，这个时候来了另外一个写操作，时间是T2，写的值是7。如果T1不是 happens before T2了，那表示可能要发生写写冲突，必须abort掉一个操作。根据逻辑时钟的规则：

    如果T1 < T2，无所谓，给数据追加一个版本就行，在T2时刻值为7。
    如果T1 > T2，根据逻辑时钟的规则，T1肯定不是 happens before T2的，也就是检测到冲突。

使用逻辑时钟，就可以检测到冲突，似乎是解决了很核心的问题，也似乎就要实现分布式事务了，其实不然。

我是想设计一个基于快照的事务模型，只要处理好了写写冲突，就可以达到快照隔离，而快照隔离就能保证不会出现脏读，不可重复读等等一些问题。但是...快照是基于一个约束的：给定一个时间戳，能够看到在这个时间戳的一个全局的快照，也就是这个时间戳之前的数据能够看到，而这个时间戳之后的看不到。

逻辑时钟因为只具备检测冲突的功能，不具备给事件界定先后顺序的能力，其实根本就无法拿到某一时刻的快照！ 也就是用某一个逻辑时钟去拿快照，得到的快照只有参与了冲突检测的那些数据，能保证一致，而全局所有的数据没这个保证的。所以其实等于拿不到快照。

没有快照，就没有快照隔离的诸多保证，事务的隔离级别里面定义的各种异常就没法保证了，这真是一个悲伤的故事。也很奇怪，数据是有多版本的，但却是没有快照的。可以检测到写写冲突、读写冲突。倒是可以考虑退化到类似于锁的方案，隔离级别方面也需要证明，要更细致的思考。

到这里，我有点气馁。某人提了一个牛B闪闪的事务模型，声称基于局部时间戳实现分布式事务，随便看下后发现作者的模型漏洞百出，没有任何关于隔离级别的讨论和证明，甚至连一致性跟隔离性的基础概念也混淆。如果一个数据库是可以读脏数据，可以读到不一致的数据，没有定义任何隔离级别，然后也算实现了一套事务模型，我只不是该呵呵哒。另一方面，这个高大上的标题启发我自己思考一下这个问题，结论是基于逻辑时钟做，局部时间戳似乎无法实现分布式事务？

当然不能止步于这里，我去研究了一下论文，还是有办法实现的，用混合逻辑时钟。先卖个关子，放到下一篇去写。想提前热身的看[这里](http://www.cse.buffalo.edu/tech-reports/2014-04.pdf)。