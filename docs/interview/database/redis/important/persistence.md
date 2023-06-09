---
title: Redis持久化策略
editLink: true
head:
  - - meta
    - name: description
      content: Redis持久化策略,包括aof,rdb及aof与rdb混合模式
layout: doc
outline: deep
---

# 持久化策略
## 概述

Redis是一个基于内存的高性能的键值型数据库，它支持三种不同的持久化策略：RDB（快照）、AOF（追加文件）、混合。这三种策略各有优缺点，需要根据不同的场景和需求进行选择和配置。本文将介绍这三种策略

## RDB（快照）

### 概述

**RDB**持久化策略是指在**一定的时间间隔内**，将**Redis**内存中的数据以**二进制文件**的形式保存到硬盘上。这个二进制文件就是一个**快照**，它记录了某个时刻**Redis**内存中的所有数据。**RDB**持久化策略可以通过配置文件或者命令来触发，配置文件中可以设置多个条件，当任意一个条件满足时，就会执行一次快照操作。如下所示：

```Bash
save 900 1 # 900秒内执行一次 set 操作 则持久化1次
save 300 10 # 300秒内执行10次 set 操作,则持久化1次
save 60 10000 # 60秒内执行10000次 set 操作,则持久化1次

```

命令有两种：

- `save`：不建议使用，会阻塞redis服务的进程，直到成功创建RDB文件
- `bgsave`：父进程创建一个子进程生成RDB文件，父进程可以正常处理客户端的指令，**不影响主进程**的服务

### 优缺点

RDB持久化策略的优点有：

- RDB文件是一个紧凑的二进制文件，占用空间小，传输速度快，适合做备份和灾难恢复
- RDB文件恢复数据的速度比AOF快，因为只需要加载一次文件即可
- RDB持久化对Redis服务器的性能影响较小，因为大部分工作由子进程完成

RDB持久化策略的缺点有：

- RDB文件不能实时或者近实时地反映Redis内存中的数据，因为它是定时触发的。如果在两次快照之间发生故障，可能会丢失一部分数据
- RDB文件在生成过程中可能会占用较多的内存和CPU资源，因为需要复制主进程的内存并执行压缩操作

## AOF（追加文件）

### 概述

**AOF**持久化策略是指将**Redis**服务器执行的每一条写命令都记录到一个文本文件中，这个文本文件就是一个**追加文件（append only file）**

AOF有三种**持久化策略**，也就是**刷盘策略**。可以根据不同的场景使用不同的**刷盘策略**。

然而随着时间的推移，AOF文件也会越来越大，因为它记录了所有的写命令。这样会导致AOF文件占用过多的磁盘空间，以及恢复数据的时间过长。为了解决这个问题，Redis提供了**AOF重写机制**，来压缩和优化AOF文件。

### 优缺点

AOF持久化策略的优点有：

- AOF文件可以实时或者近实时地记录Redis内存中的数据，因为它是每次写命令或者每秒钟同步一次。如果在同步之间发生故障，可能会丢失一部分数据，但是数据丢失的概率比RDB小。
- AOF文件是一个文本文件，可以方便地查看和编辑。AOF文件中的命令是Redis协议格式的，可以直接用Redis客户端来执行。
- AOF文件可以自动进行重写，以减少冗余命令和文件体积。重写过程不影响Redis服务器的正常服务，也不会丢失任何数据。

AOF持久化策略的缺点有：

- AOF文件通常比RDB文件大，占用更多的磁盘空间
- AOF文件恢复数据的速度比RDB慢，因为需要重新执行所有的命令
- AOF文件在写入过程中可能会出现数据不一致的情况，例如命令只写入了一半或者写入了错误的命令。这种情况下需要用redis-check-aof工具来修复AOF文件

### AOF刷盘策略

当Redis重启时，可以通过重新执行追加文件中的命令来恢复数据。AOF持久化策略可以通过配置文件来开启和设置，它决定了写命令记录到AOF文件的频率。有三个选项：

- no：写入缓存，什么时候刷盘由redis决定
- everysec：每隔一秒刷一次盘
- always：写入缓存时同时写入磁盘（尽快刷盘，而不是实时刷盘）

以下是三个策略的对比：

||||
|-|-|-|
|类型|数据安全性|性能|
|no|低|高|
|everysec|较高|较高|
|always|高|低|


### AOF重写

AOF重写机制的原理是：Redis会创建一个新的AOF文件，然后根据内存中的当前数据状态，生成相应的写命令，并写入到新的AOF文件中。这样新的AOF文件就只包含了最终数据的写命令，而不包含任何无效或者冗余的命令。例如：

```Bash
# 原始AOF文件
set a 1
set b 2
incr a
del b
set c 3

# 重写后的AOF文件
set a 2
set c 3
```

上图就是重写前和重写后的文件对比，因为AOF是追加的，是顺序读写（ES也是这样的），所以重写后的命令`set a 1`与`incr a`变成为`set a 2`。为了保证在AOF重写期间的新数据不丢失，**Redis**中引入了**AOF**重写缓冲区。当开始执行**AOF**文件重写之后又接收到客户端的请求命令，不但要将命令写入原本的**AOF**缓冲区（根据上面提到的参数刷盘），还要同时写入**AOF**重写缓冲区：

![](https://secure2.wostatic.cn/static/sH2Ncnf2Vc3WRoQQWrk8Q/redis_aof_rewrite.png?auth_key=1684286048-feAWJs15rwF6vcevzJg77u-0-b5ee64f632fc1b396eb189fd041deb46)

一旦子进程完成了**AOF**文件的重写，此时会向父进程发出信号，父进程收到信号之后会进行阻塞（阻塞期间不执行任何命令），并进行以下两项工作：

- 将**AOF**重写缓冲区的文件刷新到新的**AOF**文件内
- 将新**AOF**文件进行改名并原子操作的替换掉旧的**AOF**文件

随后，在完成了上面的两项工作之后，整个**AOF**重写工作完成，父进程开始正常接收命令。

- 自动触发：自动触发可以通过以下参数进行设置。

```.properties
# 文件大小超过上次AOF重写之后的文件的百分比。默认100
# 也就是默认达到上一次AOF重写文件的2倍之后会再次触发AOF重写
auto-aof-rewrite-percentage 100
# 设置允许重写的最小AOF文件大小,默认是64M
# 主要是避免满足了上面的百分比，但是文件还是很小的情况。
auto-aof-rewrite-min-size 64mb
```
- 手动触发：执行`bgrewriteaof`命令。

## 选取正确的持久化策略

Redis现有的持久化策略有三种：

- AOF
- RDB
- AOF与RDB混合

他们各有优缺点，需要结合不同的应用场景综合考虑，首先先讲解**AOF**和**RDB**的选择，再讲解**混合模式**

### AOF和RDB的选择

在Redis中，AOF和RDB两种持久化方式各有优缺点，一般来说，有以下几个方面需要参考：

- 数据安全性：如果要求数据不丢失，推荐**AOF**
    - **AOF**可以采取**每秒同步一次**数据或**每次写操作都同步**用来保证数据安全性
        - 如果使用**每秒同步一次**策略，则最多丢失一秒的数据
        - 如果使用**每次写操作都同步**策略，安全性达到了极致，但这会**影响性能**
    - **RDB**是一个全量的二进制文件，恢复时只需要加载到内存即可，但是可能会丢失最近几分钟的数据（取决于RDB持久化策略）
- 数据恢复速度：如果要求快速恢复数据，推荐**RDB**
    - **AOF**需要重新执行所有的写命令，恢复时间会更长
    - **RDB**是一个全量的二进制文件，恢复时只需要加载到内存即可
- 数据备份和迁移：如果要求方便地进行数据备份和迁移，推荐**RDB**
    - **AOF**文件可能会很大，传输速度慢
    - **RDB**文件是一个紧凑的二进制文件，占用空间小，传输速度快
- 数据可读性：如果要求能够方便地查看和修改数据，推荐**AOF**
    - **AOF**是一个可读的文本文件，记录了所有的写命令，可以用于灾难恢复或者数据分析
    - **RDB**是一个二进制文件，不易查看和修改

||数据安全性|数据恢复速度|数据备份和迁移|数据可读性|
|-|-|-|-|-|
|AOF|高|低|低|高|
|RDB|低|高|高|低|


### AOF与RDB的混合模式

综合上一节，我们可以根据不同的场景和需求来选择合适的持久化方式。但是，在实际应用中，并不一定要二选一，也可以**同时使用AOF和RDB两种持久化方式**。这样可以利用AOF来保证数据不丢失，作为数据恢复的第一选择；用RDB做不同程度的冷备份，当AOF备份文件丢失或损坏不可用时，可以使用RDB快照文件快速地恢复数据

综上所述，混合模式兼并了RDB重启后的快速恢复能力和AOF丢失数据风险低的能力，具体操作流程如下：

1. 子进程会通过`BGSAVE `写入AOF中
2. 触发`BGREWRITEAOF`后，会将AOF写入到文件
3. 将含有RDB和AOF的数据覆盖旧的AOF文件（这时AOF文件一半为RDB，一半为AOF）

混合模式的AOF文件：

```text
REDIS0008?redis-ver4.0.1?redis-bits繞?ctime聮~`?used-mem?? ?aof-preamble??repl-id(6c3378899b63bc4ebeaafaa09c27902d514eeb1f?repl-offset??? list1?77   /   appleorangegrape?e k1v1彝髖S[zb*2
$6
SELECT
$1
0
*3
$4
sadd
$8
gamedisk
$4
nioh
*3
$4
sadd
$8
gamedisk
$4
tomb
```

如果想要开启混合模式，在`redis.conf`中配置：

```.properties
aof-use-rdb-preamble yes
```

同时使用AOF和RDB两种持久化方式也需要注意一些问题：

- AOF重写和RDB持久化可能会同时发生冲突，导致内存、CPU和磁盘的消耗增加。为了解决这个问题，Redis采用了一些策略来协调两者之间的关系。具体可以参考下面的介绍（**AOF重写和RDB持久化的冲突**）
- AOF文件可能会变得很大，导致磁盘空间不足或者恢复时间过长。为了解决这个问题，Redis提供了AOF重写机制来压缩AOF文件。具体可以参考上一节（**AOF重写**）
- AOF文件可能会被损坏或者丢失，导致数据无法恢复。为了解决这个问题，Redis提供了AOF校验机制来检测AOF文件是否完整。具体可以参考下面的介绍（**AOF校验机制**）

### AOF重写和RDB持久化的冲突

在Redis中，AOF重写和RDB持久化可能会同时发生，这会导致一些冲突和问题。例如：

- AOF重写和RDB持久化都需要fork子进程，如果两个子进程同时存在，会增加内存的消耗和系统的负载。
- AOF重写和RDB持久化都需要写入磁盘，如果两个文件同时写入，会增加磁盘的压力和IO的开销。
- AOF重写和RDB持久化都需要在完成后通知主进程，如果两个信号同时到达，可能会造成信号丢失或者处理错误。

为了解决这些冲突和问题，Redis采用了以下策略：

- 如果AOF重写和RDB持久化同时被触发，那么只有一个子进程会被创建，优先执行RDB持久化，然后再执行AOF重写。这样可以避免同时存在两个子进程的情况。
- 如果AOF重写正在进行，而此时又收到了RDB持久化的请求，那么RDB持久化会被延迟到AOF重写完成后再执行。这样可以避免同时写入两个文件的情况。
- 如果AOF重写和RDB持久化都完成了，那么主进程会先处理RDB持久化的信号，然后再处理AOF重写的信号。这样可以避免信号丢失或者处理错误的情况。

总之，Redis通过优先级、延迟和顺序等方式来协调AOF重写和RDB持久化的冲突和问题，保证了数据的完整性和一致性，下图为简要说明。

|场景|策略|
|-|-|
|AOF重写与RDB持久化同时被触发|优先RDB|
|AOF重写正在进行|优先AOF|
|AOF重写和RDB持久化都完成|优先RDB|


### AOF校验机制

AOF校验机制是指在Redis启动时，对AOF文件进行检查，判断文件是否完整，是否有损坏或者丢失的数据。如果发现AOF文件有问题，Redis会拒绝启动，并给出相应的错误信息

AOF校验机制的原理是使用一个64位的校验和（checksum）来对AOF文件进行验证。校验和是一个数字，它是根据AOF文件的内容计算出来的，如果AOF文件的内容发生了任何改变，那么校验和也会发生变化。因此，通过比较计算出来的校验和和保存在AOF文件末尾的校验和，就可以判断AOF文件是否完整。

具体来说，AOF校验机制的过程如下：

- 当Redis执行AOF重写时，它会在新的AOF文件末尾写入一个特殊的命令：`*1\r\n$6\r\nCHECKSUM\r\n`，这个命令表示接下来要写入一个校验和
- Redis会使用CRC64算法，对新的AOF文件中除了最后一行之外的所有内容进行计算，得到一个64位的数字作为校验和，并将这个数字以16进制的形式写入到新的AOF文件末尾。
- Redis会将新的AOF文件替换旧的AOF文件，并将校验和保存在内存中
- 当Redis重启时，它会读取AOF文件，并使用同样的CRC64算法，对除了最后一行之外的所有内容进行计算，得到一个64位的数字作为校验和，并将这个数字与内存中保存的校验和进行比较
- 如果两个校验和相同，说明AOF文件没有损坏或者丢失数据，Redis会继续启动并加载AOF文件中的数据
- 如果两个校验和不同，说明AOF文件有问题，Redis会拒绝启动，并给出类似于`Bad file format reading the append only file: checksum mismatch`这样的错误信息

通过这种方式，Redis可以保证在启动时检测到AOF文件是否完整，从而避免加载错误或者不完整的数据。当然，这种机制也有一些局限性：

- AOF校验机制只能在Redis启动时执行，如果在运行过程中AOF文件被修改或者损坏，Redis**无法及时发现**。
- AOF校验机制只能检测到**AOF文件是否完整**，但不能检测到**AOF文件是否正确**。比如说，如果有人恶意地修改了AOF文件中的某些命令或者参数，导致数据逻辑上出现错误，那么Redis无法识别出这种情况。
- **AOF校验机制会增加Redis启动时的时间开销**，因为需要对整个AOF文件进行计算。如果AOF文件很大，那么这个过程可能会很慢。

总之，AOF校验机制是一种简单而有效的方法，可以保证在Redis启动时检测到AOF文件是否完整。但是它也有一些局限性和代价，需要在实际应用中权衡利弊。

### 三种模式的选择建议

具体的选择建议如下：

- 如果对数据完整性要求不高，可以只使用RDB，或者将AOF的同步频率设置为每秒一次
- 如果想让数据尽可能不丢失，可以只使用AOF，并将AOF的同步频率设置为每次写入操作都同步
- 如果对数据完整性和性能都有要求，可以同时使用AOF和RDB，并将AOF的同步频率设置为每秒一次。这样既可以保证数据的安全性，又可以利用RDB进行快速的数据恢复
- 如果既想节省磁盘空间，又想提高数据恢复速度，可以只使用RDB，并适当调整RDB的快照频率

AOF和RDB两种持久化方式各有优缺点，需要根据具体的场景和需求来进行选择和配置。在选择时，需要考虑以下几个因素：

- 数据完整性：即数据丢失的风险和可接受的范围
- 数据恢复速度：即从持久化文件恢复到内存中所需的时间
- 磁盘空间占用：即持久化文件所占用的磁盘空间大小
- 写入性能：即持久化操作对Redis服务端的写入性能的影响

> 注意:
AOF策略设置为 always 或 everysec，并且`BGSAVE `或`BGREWRITEAOF`正在对磁盘执行大量 I/O 时，Redis 刷盘可能会阻塞
可以设置`no-appendfsync-on-rewrite yes`，来缓解这个问题。这样的话，当另一个子进程正在保存的时候，Redis 的持久性与`appendfsync no`相同。实际上，最严重的情况是丢失30秒的日志

## 持久化策略常见问题及解决方案

### AOF文件过大

当AOF文件过大时，会占用磁盘空间，影响写入性能，甚至导致Redis启动失败。可以使用`bgrewriteaof`命令或者配置`auto-aof-rewrite-percentage`和`auto-aof-rewrite-min-size`参数来触发AOF重写操作，将AOF文件压缩为最小的命令集合

```Bash
# 文件大小超过上次AOF重写之后的文件的百分比。默认100
# 也就是默认达到上一次AOF重写文件的2倍之后会再次触发AOF重写
auto-aof-rewrite-percentage 100
# 设置允许重写的最小AOF文件大小,默认是64M
# 主要是避免满足了上面的百分比，但是文件还是很小的情况。
auto-aof-rewrite-min-size 64mb
```

### AOF文件损坏

当AOF文件损坏时，会导致Redis无法正常启动或者恢复数据。可以使用`redis-check-aof`工具来修复AOF文件，或者使用备份的RDB文件来恢复数据

### AOF 文件可能会被截断

在 Redis 启动过程中，当 AOF 数据被加载回内存时，可能会发现 AOF 文件在最后被截断

- `aof-load-truncated yes`，则加载截断的 AOF 文件，并且记录日志
- `aof-load-truncated no`，则服务器会因错误拒绝启动，且需要在启动服务器之前使用`redis-check-aof`修复aof文件

可以在`redis.conf`中配置：

```.properties
aof-load-truncated yes
```

**可记录时间戳帮助恢复数据**

如果在**AOF**记录时间戳，可能会与现有的**AOF**解析器不兼容，默认关闭

`redis.conf`中配置：

```.properties
aof-timestamp-enabled no
```

### RDB文件丢失

当RDB文件丢失时，会导致Redis无法恢复数据。为了解决这个问题，可以使用备份的AOF文件或者其他节点的RDB文件来恢复数据，或者增加RDB的快照频率来减少数据丢失的风险

### RDB文件损坏

当RDB文件损坏时，会导致Redis无法恢复数据。为了解决这个问题，可以使用`redis-check-rdb`工具来检查和修复RDB文件，或者使用备份的AOF文件或者其他节点的RDB文件来恢复数据