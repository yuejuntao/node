[TOC]

### 一  回顾一条SQL执行流程

之前我们已经分析了MySQL架构上的整体设计原理，现在对一条SQL语句从我们的系统层面发送到MySQL中，然后一步一步执行这条SQL的流程，都有了一个整体的了解。

我们已经知道了，MySQL最常用的就是InnoDB存储引擎，那么我们今天借助一条更新语句的执行，来初步的了解一下InnoDB存储引擎的架构设计。

首先假设我们有一条SQL语句是这样的：

````mysql
update users set name='xxx' where id=10
````

那么我们先想一下这条SQL语句是如何执行的？

首先肯定是我们的系统通过一个数据库连接发送到了MySQL上，然后肯定会经过SQL接口、解析器、优化器、执行器几个环节，解析SQL语句，生成执行计划，接着去由执行器负责这个计划的执行，调用InnoDB存储引擎的接口去执行。

### 二  InnoDB的重要内存结构：缓冲池（Buffer Pool）

InnoDB存储引擎中有一个非常重要的放在内存里的组件，就是缓冲池（Buffer Pool），这里面会缓存很多的数据，以便于以后在查询的时候，万一你要是内存缓冲池里有数据，就可以不用去查磁盘了。

<img src="http://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201102213052109.png?token=ACGZFEEAKJPP2ALQQFSJ4X27UAFE2" alt="image-20201102213052109" style="zoom:33%;" />

存储引擎要执行更新语句的时候 ，比如对“id=10”这一行数据，他其实会先将“id=10”这一行数据看看是否在缓冲池里，如果不在的话，那么会直接从磁盘里加载到缓冲池里来，而且接着还会对这行记录加独占锁。因为我们想一下，在我们更新“id=10”这一行数据的时候，肯定是不允许别人同时更新的，所以必须要对这行记录加独占锁。

### 三  Undo Log用于回滚

接着下一步，假设“id=10”这行数据的name原来是“zhangsan”，现在我们要更新为“xxx”，那么此时我们得先把要更新的原来值“zhangsan”和“id=10”这些信息，写入到`undo`日志文件中去。

其实稍微对数据库 有一点了解的同学都应该知道，如果我们执行一个更新语句，要是他是在一个事务里的话，那么事务提交之前我们都是可以对数据进行回滚的，也就是把你更新为“xxx”的值回滚到之前的“zhangsan”去。

所以为了考虑到未来可能要回滚数据的需要，这里会把你更新前的值写入undo日志文件，我们看下图

<img src="http://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201102213313076.png?token=ACGZFEFAYVPSK5O7MERHMJS7UAFNS" alt="image-20201102213313076" style="zoom:33%;" />

### 四  更新Buffer Pool中缓存的数据

当我们把要更新的那行记录从磁盘文件加载到缓冲池，同时对他加锁之后，而且还把更新前的旧值写入undo日志文件之后，我们就可以正式开始更新这行记录了，更新的时候，先是会更新缓冲池中的记录，此时这个数据就是`脏数据`了。

这里所谓的更新内存缓冲池里的数据，意思就是把内存里的“id=10”这行数据的name字段修改为“xxx”，那么为什么说此时这行数据就是脏数据了呢？

因为这个时候磁盘上“id=10”这行数据的name字段还是“zhangsan”，但是内存里这行数据已经被修改了，所以就会叫他是脏数据。

我们看下图，我同时把几个步骤的序号标记出来了。

<img src="http://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201102213541172.png?token=ACGZFECCWY5KSH5VXZYPIMK7UAFW6" alt="image-20201102213541172" style="zoom:33%;" />

### 五  Redo Log Buffer：万一系统宕机，如何避免数据丢失？

#### 1  更新Buffer Pool后写入Redo Log Buffer

按照上图的说明，现在已经把内存里的数据进行了修改，但是磁盘上的数据还没修改，那么此时万一MySQL所在的机器宕机了，必然会导致内存里修改过的数据丢失，这可怎么办呢？

这个时候，就必须要把对内存所做的修改写入到一个**Redo Log Buffer**里去，这也是内存里的一个缓冲区，是用来存放redo日志的。

所谓的redo日志，就是记录下来你对数据做了什么修改，比如对**id=10这行记录修改了name字段的值为xxx**，这就是一个日志。

<img src="http://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103213131019.png" alt="image-20201103213131019" style="zoom: 33%;" />

这个redo日志其实是用来在MySQL突然宕机的时候，用来恢复你更新过的数据的，但是我们现在还没法直接讲解redo log是如何使用的，毕竟现在redo日志还仅仅停留在内存缓冲里。

以我们都知道，其实在数据库中，哪怕执行一条SQL语句，其实也可以是一个独立的事务，只有当你提交事务之后，SQL语句才算执行结束。

所以这里我们都知道，到目前为止，其实还没有提交事务，那么此时如果MySQL崩溃，必然导致内存里Buffer Pool中的修改过的数据都丢失，同时你写入Redo Log Buffer中的redo日志也会丢失：

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103213354049.png" alt="image-20201103213354049" style="zoom:33%;" />

其实这个时候数据丢失是不要紧的，因为你一条更新语句，没提交事务，就代表他没执行成功，此时MySQL宕机虽然导致内存里的数据都丢失了，但是你会发现，磁盘上的数据依然还停留在原样子。

也就是说，“id=1”的那行数据的name字段的值还是老的值，“zhangsan”，所以此时你的这个事务就是执行失败了，没能成功完成更新，你会收到一个数据库的异常。然后当mysql重启之后，你会发现你的数据并没有任何变化。

所以此时如果mysql宕机，不会有任何的问题。

#### 2  提交事务的时候将redo日志写入磁盘中

接着我们想要提交一个事务了，此时就会根据一定的策略把redo日志从redo log buffer里刷入到磁盘文件里去。

此时这个策略是通过 **innodb_flush_log_at_trx_commit** 来配置的，他有几个选项。

1. 当参数的值为0的时候，那么你提交事务的时候，不会把redo log buffer里的数据刷入磁盘文件的。此时可能你都提交事务了，结果mysql宕机了，然后此时内存里的数据全部丢失。相当于你提交事务成功了，但是由于MySQL突然宕机，导致内存中的数据和redo日志都丢失了。

2. 当参数的值为1的时候，你提交事务的时候，就必须把redo log从内存刷入到磁盘文件里去，只要事务提交成功，那么redo log就必然在磁盘里了，我们看下图：

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103213906966.png" alt="image-20201103213906966" style="zoom:33%;" />

这时，如果mysql系统突然崩溃了，是不会丢失数据的。因为虽然内存里的修改成name=xxx的数据会丢失，但是redo日志里已经说了，对某某数据做了修改name=xxx。所以此时mysql重启之后，他可以根据redo日志去恢复之前做过的修改。

3. 当参数的值是2时，提交事务的时候，把redo日志写入磁盘文件对应的os cache缓存里去，而不是直接进入磁盘文件，可能1秒后才会把os cache里的数据写入到磁盘文件里去。

   这种模式下，你提交事务之后，redo log可能仅仅停留在os cache内存缓存里，没实际进入磁盘文件，万一此时你要是机器宕机了，那么os cache里的redo log就会丢失，同样会让你感觉提交事务了，结果数据丢了，看下图。

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103214217598.png" alt="image-20201103214217598" style="zoom: 50%;" />

**innodb_flush_log_at_trx_commit** 通常建议是设置为1，即提交事务的时候，redo日志必须是刷入磁盘文件里的。

这样可以严格的保证提交事务之后，数据是绝对不会丢失的，因为有redo日志在磁盘文件里可以恢复你做的所有修改。

如果要是选择0的话，可能你提交事务之后，mysql宕机，那么此时redo日志没有刷盘，导致内存里的redo日志丢失，你提交的事务更新的数据就丢失了；如果要是选择2的话，如果机器宕机，虽然之前提交事务的时候，redo日志进入os cache了，但是还没进入磁盘文件，此时机器宕机还是会导致os cache里的redo日志丢失。

所以对于数据库这样严格的系统而言，一般建议redo日志刷盘策略设置为1，保证事务提交之后，数据绝对不能丢失。

### 六  MySQL binlog

我们之前说的redo log，他是一种偏向物理性质的重做日志，因为他里面记录的是类似这样的东西，“**对哪个数据页中的什么记录，做了个什么修改**”。而且redo log本身是属于InnoDB存储引擎特有的一个东西。

而binlog叫做归档日志，他里面记录的是偏向于逻辑性的日志，类似于“**对users表中的id=10的一行数据做了更新操作，更新以后的值是什么**”。binlog不是InnoDB存储引擎特有的日志文件，是属于mysql server自己的日志文件。

#### 1  提交事务的时候，同时会写入binlog

在我们提交事务的时候，会把redo log日志写入磁盘文件中去。然后其实在提交事务的时候，我们同时还会把这次更新对应的binlog日志写入到磁盘文件中去，如下图所示。

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103214631920.png" alt="image-20201103214631920" style="zoom: 33%;" />

执行器会负责跟InnoDB进行交互，包括从磁盘里加载数据到Buffer Pool中进行缓存，包括写入undo日志，包括更新Buffer Pool里的数据，以及写入redo log buffer，redo log刷入磁盘，写binlog，等等。

实际上，执行器是非常核心的一个组件，负责跟存储引擎配合完成一个SQL语句在磁盘与内存层面的全部数据更新操作。

而且我们在上图可以看到，我把一次更新语句的执行，拆分为了两个阶段：

1. 上图中的1、2、3、4几个步骤，其实本质是执行这个更新语句的时候干的事。

2. 上图中的5和6两个步骤，是从提交事务开始的，属于提交事务的阶段了。

#### 2  binlog日志的刷盘策略分析

对于binlog日志，其实也有不同的刷盘策略，有一个 **sync_binlog** 参数可以控制binlog的刷盘策略，他的默认值是0，

此时你把binlog写入磁盘的时候，其实不是直接进入磁盘文件，而是进入os cache内存缓存。所以跟之前分析的一样，如果此时机器宕机，那么你在os cache里的binlog日志是会丢失的，我们看下图的示意

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103214928471.png" alt="image-20201103214928471" style="zoom: 33%;" />

如果要是把**sync_binlog**参数设置为1的话，那么此时会强制在提交事务的时候，把binlog直接写入到磁盘文件里去，那么这样提交事务之后，哪怕机器宕机，磁盘上的binlog是不会丢失的，如下图所示

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103215015699.png" alt="image-20201103215015699" style="zoom:33%;" />

####  3  基于binlog和redo log完成事务的提交

当我们把binlog写入磁盘文件之后，接着就会完成最终的事务提交，此时会把本次更新对应的binlog文件名称和这次更新的binlog日志在文件里的位置，都写入到redo log日志文件里去，同时在redo log日志文件里写入一个commit标记。在完成这个事情之后，才算最终完成了事务的提交，我们看下图的示意。

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103215126923.png" alt="image-20201103215126923" style="zoom:33%;" />

最后在redo日志中写入commit标记其实是用来保持redo log日志与binlog日志一致的。

我们来举个例子，假设我们在提交事务的时候，一共有上图中的5、6、7三个步骤，必须是三个步骤都执行完毕，才算是提交了事务。那么在我们刚完成步骤5的时候，也就是redo log刚刷入磁盘文件的时候，mysql宕机了，此时怎么办？

这个时候因为没有最终的事务commit标记在redo日志里，所以此次事务可以判定为不成功。不会说redo日志文件里有这次更新的日志，但是binlog日志文件里没有这次更新的日志，不会出现数据不一致的问题。

如果要是完成步骤6的时候，也就是binlog写入磁盘了，此时mysql宕机了，怎么办？

同理，因为没有redo log中的最终commit标记，因此此时事务提交也是失败的。

必须是在redo log中写入最终的事务commit标记了，然后此时事务提交成功，而且redo log里有本次更新对应的日志，binlog里也有本次更新对应的日志 ，redo log和binlog完全是一致的。

### 七  后台IO线程随机将内存更新后的脏数据刷回磁盘

现在我们假设已经提交事务了，此时一次更新“update users set name='xxx' where id=10”，他已经把内存里的buffer pool中的缓存数据更新了，同时磁盘里有redo日志和binlog日志，都记录了把我们指定的“id=10”这行数据修改了“name='xxx'”。

但是这个时候磁盘上的数据文件里的“id=10”这行数据的name字段还是等于zhangsan这个旧的值啊！所以MySQL有一个后台的IO线程，会在之后某个时间里，随机的把内存buffer pool中的修改后的脏数据给刷回到磁盘上的数据文件里去，我们看下图：

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103215353854.png" alt="image-20201103215353854" style="zoom: 50%;" />

在你IO线程把脏数据刷回磁盘之前，哪怕mysql宕机崩溃也没关系，因为重启之后，会根据redo日志恢复之前提交事务做过的修改到内存里去，就是id=10的数据的name修改为了xxx，然后等适当时机，IO线程自然还是会把这个修改后的数据刷到磁盘上的数据文件里去。

### 八  总结

InnoDB存储引擎主要就是包含了一些buffer pool、redo log buffer等内存里的缓存数据，同时还包含了一些undo日志文件，redo日志文件等东西，同时mysql server自己还有binlog日志文件。

在你执行更新的时候，每条SQL语句，都会对应修改buffer pool里的缓存数据、写undo日志、写redo log buffer几个步骤；

但是当你提交事务的时候，一定会把redo log刷入磁盘，binlog刷入磁盘，完成redo log中的事务commit标记；最后后台的IO线程会随机的把buffer pool里的脏数据刷入磁盘里去。

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201103215830575.png" alt="image-20201103215830575" style="zoom:50%;" />