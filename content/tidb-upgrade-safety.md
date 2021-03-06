如何不考虑兼容性，都好办。但是考虑已经有上线用户，就比较麻烦。必须考虑兼容性，能够平滑升级，不影响到业务。

除了数据编码外，还有一些其它可能有兼容性的地方。比如某个全局session variable，以前默认值是1，现在新版本里面要变成2。比如某张表以前没有，现在新版本想添加。比如以前有某张表，现在想修改schema添加一列...

现在的做法，TiDB里面存了一个字段表示当前的TiDB的bootstrap版本，在bootstrap的时候如果发现程序的版本大于当前bootstrap的版本，则执行upgrade操作。

由于TiDB是分布式的，升级版本会引入一些风险性：

* 多台机器同时执行bootstrap的一致性保证

假设整个集群同时重启，会有多个机器同时执行bootstrap操作。需要保证只有一个做bootstrap的TiDB能够成功，其它都会失败。执行失败的TiDB也能正确启动，不会起不来。

涉及到DDL操作不会因为同时执行而被重复执行多次。失败的重试不会用旧版本数据覆盖新的。更新的操作和版本号不会交叉，即更新操作成功，修改版本号却失败了，或者更新失败而版本号修改了。

* 升级过程中断了是否可恢复

如果升级过程中，被用户手动杀进程了，整个数据应该处于可恢复的状态。不能出现，那bootstrap处于一个升级版本做了一半的状态。
然后再执行一直失败，永远处于一个再也无法启动的状态，整个集群就废了。

* DDL的花费的时间问题

实现上的限制，做DDL添加列每次只能添加1列。每个DDL最快要经过两个lease。如果lease时间设置的10秒，如果更新涉及30个DDL操作，那升级操作会花费10分钟。

* slave集群的存在对升级的影响

如果有slave集群存在，会有binlog同步的过程。binlog同步在遇到DDL会阻塞。slave会被阻塞这也是一个风险。

主要说说多机同时bootstrap的一致性问题。这里想到三种方案

第一种，最简单直观的应该是分布式锁。假设有分布式锁的机制，就可以保证只有一个机器拿到锁，只有拿到锁的那台机器能做bootstrap，其它都拿无法执行bootstrap。

分布式锁的，这里有一篇同事的[demo教程](http://0xffff.me/shi-yong-tikv-zuo-wei-fen-bu-shi-suo/)。

第二种，利用事务实现。

TiDB本身已经有事务了。如果把更新要做的操作，和更新版本号，扔到一个事务里面，就可以保证两者都成功，或者两者都失败。

    begin
    updateOperation
    updateVersion
    commit

但是这种遇到updateOperation里面有DDL的时候会出现问题，因为DDL会自动提交前面的操作，就不能保证整个是在一个事务了。

第三种，把整个过程拆小，并保证每一小步操作都是可重入的，记录checkpoint。我觉得这种理解上面最复杂，展开说一下。

关键点在于可重入。如果是加表需要是create table if not exist，执行多次也没问题。如果插入数据，则需要把插数据和改版本放事务里，不能让插数据执行多次。

然后保证：checkpoint之前的，都是一定成功了的。也就是操作成功才记录checkpoint，而如果操作失败，即使有脏数据，由于是可重入的，从上个checkpoint开始重新执行一遍，也没问题。

以上就是TiDB在版本升级时的一些实现细节。
