---
title: CockroachDB分布式事务解析
tags:
  - 技术
categories:
  - 技术
  - CockroachDB
  - CockroachDB 架构解析
cover: 'https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/20.jpeg'
abbrlink: 9458b181
date: 2021-06-28 22:13:18
---

# 事务层

事务层实现了对并发操作的ACID事务支持。

* CRDB事务分为两个阶段：

  * write & reads，即事务执行阶段，当进行写操作时，CRDB并不会直接对硬盘的数据进行修改，而是使用另外两个东西来进行辅助，这两个东西也避免了锁的使用：
![transaction-execution-process.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/cockroach/txn/transaction-execution-process.png)
  * Transaction record，存储在range第一个被修改的key处，表明了修改当前key的事务所处的状态：PENDING，COMMITTED或者ABORTED。第一个状态表示事务正在进行；第二个状态表示事务已经提交；第三个表示事务已经丢弃，正在进行重试或者rollback。

* Write intents，对数据的修改写在这里，此外还带一个指向transaction record的指针，来表明这些数据当前或者之前被一个事务修改过，其他事务根据这个状态来决定对这些数据的下一步操作。产生一个新的write intent前需要检查这一块数据有没有更晚时间戳的提交值，如果有的话该事务需要被重新开始。

当进行读操作时，会检查所读的每块数据的write intent，如果不存在，那么直接读以前的mvcc数据就行了，如果有write intent，那么需要判断intent的状态，根据状态做出下一步操作。

* Commit，提交阶段，如果事务执行阶段没有问题，那么事务就可以直接进行提交，如果中途出现问题就会abort，然后进行重试或者rollback。

* Cleanup，这个阶段不属于事务中，主要是用来解决事务中产生的write intent。intent并不是事务结束后就立即进行处理，而是异步处理，当有新的read或者write操作这一块数据的时候会发现这些intent，这时候如果状态是commit，那么就会把intent指向record的指针删除将其变成普通的mvcc数据，如果是aborted，那么直接将intent删除。

**Detailed design ：**

每个事务在协调节点开始时都会有一个candidate timestamp，如果事务中途执行过程中不改变，那么这个就是mvcc中用来标志事务的timestamp，当隔离等级为snapshot时，中途可能会push事务的timestamp，从而改变candidate timestamp，最后提交的时候多个node的timestamp可能会不一致，这时候取最大的那个。 timestamp由HLC产生，hlc由物理时钟和逻辑时钟组成。物理时钟l.j是每个node的本地wall time由NTP产生，逻辑时钟c.j初始值为0，每当碰到相同的物理时间时会将其中一个逻辑时钟加一从而形成区别，因而比较hlc时间时首先比较物理时钟然后比较逻辑时钟。每当有一个event的时候hlc的时间都会更新：

```go
l'.j = l.j; // 跟当前系统时间比较，得到pt

l.j = max(l'.j, pt.j) // 如果pt没有变化，则c.j加1，如果有变化，因为这时候

// 铁定PT变大了，所以我们可以将ct清零

if (l.j = l'.j)

{ c.j = c.j + 1 }

else { c.j = 0 }

// Timestamp with l.j, c.j
```

```go
当有一个本地event发生时，会比较当前hlc的物理时间跟系统的物理时间，取其中最大的，如果相等的话则将逻辑时间加一。为了减少节点间的时间偏移，每当其他node的event到来的时候也会更新hlc时间：
  l'.j = l.j;
// 跟当前系统事件以及节点m的pt比较，得到pt
  l.j = max(l'.j, l.m, pt.j)
  if (l.j = l'.j = l.m) {
    // pt一样，获取最大的ct，并加1
    c.j = max(c.j, c.m) + 1
  } else if (l.j = l'j) {
    // 这里表明j原来的pt比m大，只需要增加ct
    c.j = c.j + 1
  } else if (l.j = l.m) {
    // 这里表明m的pt比j原来的要大，所以直接可以用m的ct + 1
    c.j = c.m + 1
  } else {
    // pt变化了，ct清零
    c.j = 0
}
// Timestamp with l.j, c.
```
会将其他node的hlc的物理时间，本node维护的hlc的物理时间和ntp的物理时间比较，将本地维护的hlc物理时间设为最大的，如果相等则将逻辑时间最大的加一。 同时为了保证事务的外部一致性，即当一个新的事务开始的时候，需要保证分配给他的candidate timestamp需要大于所有他接下来操作涉及数据的timestamp。为了做到这点crdb给新事物分配candidate timestamp时都会给其定一个时间偏移，时间偏移是当前node与cluster的最大时间偏移，在candidate timestamp到之后时间偏移这段时间称为不确定时间，即如果碰到了在这段时间修改的数据，不能判断其timestamp到底是在candidate timestamp之前还是之后，当事务执行过程中在任意node碰到这种情况，事务就会restart。为了避免因为这种情况导致事务不停的restart，在restart之后的candidate timestamp会push到不确定数据的timestamp，而时间偏移的上限不变，这样就减少了时间偏移的间隔，从而减少了restart的可能性。 跟Google的True time相比，这个优点是不需要google那样提供精确的GPS和原子种，类似的点是都可以在本地生成时间戳而不需要一个时间中心之类的辅助，缺点是生成的时间戳的不准确性需要在事务执行时才能发现，会导致事务restart，而Google通过commit wait避免了这个情况。

* 隔离等级：crdb有两种隔离等级，SNAPSHOT和SERIALIZABLE，第二种是默认的隔离等级，这两种等级在实现上的主要区别在于两个事务在相同数据intent处发生冲突时，前者可以向后push其中一个事务的timestamp。crdb这两种隔离等级的实现都不需要锁，它们依赖于前面提到的transaction record和write intent，通过它们可以发现冲突并选择解决方法，不过代价就是更多的事务重试。

* 事务执行流程细节：

  * 事务开始后，先选取一个range写入transaction record状态PENDING，与此同时将所有需要修改的数据写入一个write intent，write intent中带普通的MVCC数据和一个指针指向record。当事务读取数据的时候，如果没有发现intent，那么只需要正常读取MVCC数据就行了，如果发现了intent，就需要根据intent的指针找到record从而确定当前正在使用intent的事务的状态，在下面我列了事务冲突的几种情况。

    * 提交事务时更新record状态，但是并不清理intent。提交事务使用的时间戳一般是candidate timestamp，但是如果是SNAPSHOT隔离等级，那么可能每个node的candidate timestamp不一致，这时协调节点会取最大的那一个。

    * 不论是事务commit了或者abort了，之前事务的intent在当时都没有处理，而是在下一次有write或者read使用这块数据的时候会发现这些intent，这时会根据状态来清理intent，如果是commit那么将其转变为MVCC数据，如果是abort那么将其删除。需要注意的一点是record状态的更新是协调节点通过heartbeat对应record来更新和确认事务存活的，因而每次事务检查record状态时都需要heartbeat协调节点来确定事务状态是否准确的。

* 冲突解决：

  * 读操作碰到时间戳较新的write intent：这不会造成冲突，因为修改在读之后，读操作直接按时间戳读取MVCC数据即可。

  * 读操作碰到时间戳相近的write intent：由于cluster的时钟存在偏移，因而这一时间段的数据状态不能确认，需要事务restart，具体的前面说过。

  * 读操作碰到时间戳较旧的write intent：通过指针查看record状态，如果事务已经提交了，那么直接读取该intent数据并将其变成MVCC数据即可；如果事务没有提交，那么根据隔离等级来进行处理：
    * 1.当隔离等级为SNAPSHOT时，为了让读操作尽快完成，这时会将写操作的timestamp往后push，这样当写操作的timestamp大于当前读的timestamp后就不会造成冲突了，但是这样也可能会造成写偏斜（write skew）。
    * 2.当隔离等级为SERIALIZABLE时，则根据事务的优先级来判断，如果读操作优先级较高，那么restart写事务；如果读操作跟写事务优先级相等或者较少，那么restart读事务并重新给一个较高的优先级（冲突事务优先级-1）。

  * 写操作碰到未提交的write intent：如果写操作的优先级较高，那么restart冲突的事务；如果相等或者较低，则restart写操作事务并给一个较高的优先级（冲突事务优先级-1）.

  * 写操作碰到最近新提交的值：由于正在进行的写操作时间戳在已提交的时间戳前面，那么不能继续执行写操作不然会导致脏数据，需要restart写操作事务，优先级不变但是candidate timestamp push到发生冲突的值的timestamp之后。

  * 写操作碰到最近被读过的值：最近读操作的时间戳会存在timestamp cache中，如果写操作的timestamp小于cache中读操作的timestamp，那么继续写就会造成问题，因而使写操作事务以一个新的timestamp restart。

* 对于一个事务它的重试的次数跟持续时间的关系：

![transaction-execution-process.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/cockroach/txn/tranction-completion.png)