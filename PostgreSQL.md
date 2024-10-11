
# 九、PostgreSQL中的表


# 十、事务

## 10.1 什么是ACID？（常识）

在日常操作中，对于一组相关操作，通常要求要么都成功，要么都失败。在关系型数据库中，称这一组操作为事务。为了保证整体事务的安全性，有ACID这一说：

* 原子性A：事务是一个最小的执行单位，一次事务中的操作要么都成功，要么都失败。
* 一致性C：在事务完成时，所有数据必须保持在一致的状态。（事务完成后吗，最终结果和预期结果是一致的）
* 隔离性：一次事务操作，要么是其他事务操作前的状态，要么是其他事务操作后的状态，不存在中间状态。
* 持久性：事务提交后，数据会落到本地磁盘，修改是永久性的。

PostgreSQL中，在事务的并发问题里，也是基于MVCC，多版本并发控制去维护数据的一致性。相比于传统的锁操作，MVCC最大的有点就是可以让 **读写互相不冲突** 。

当然，PostgreSQL也支持表锁和行锁，可以解决写写的冲突问题。

PostgreSQL相比于其他数据，有一个比较大的优化，DDL也可以包含在一个事务中。比如集群中的操作，一个事务可以保证多个节点都构建出一个表，才算成功。

## 10.2 事务的基本使用

首先基于前面的各种操作，应该已经体会到了，PostgreSQL是自动提交事务。跟MySQL是一样的。

可以基于关闭PostgreSQL的自动提交事务来进行操作。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/e911b0468569428b863b5395fc8dd2b0.png)

但是上述方式比较麻烦，传统的方式。

就是三个命令：

* begin：开始事务
* commit：提交事务
* rollback：回滚事务

```sql
-- 开启事务
begin;
-- 操作
insert into test values (7,'bbb',12,5);
-- 提交事务 
commit;
```

## 10.3 保存点（了解）

比如项目中有一个大事务操作，不好控制，超时有影响，回滚会造成一切重来，成本太高。

我针对大事务，拆分成几个部分，第一部分完成后，构建一个保存点。如果后面操作失败了，需要回滚，不需要全盘回滚，回滚到之前的保存点，继续重试。

有人会发现，破坏了整体事务的原子性。

But，只要操作合理，可以在保存点的举出上，做重试，只要重试不成功，依然可以全盘回滚。

比如一个电商项目，下订单，扣库存，创建订单，删除购物车，增加用户积分，通知商家…………。这个其实就是一个大事务。可以将扣库存和下订单这种核心功能完成后，增加一个保存点，如果说后续操作有失败的，可以从创建订单成功后的阶段，再做重试。

不过其实上述的业务，基于最终一致性有更好的处理方式，可以保证可用性。

简单操作一下。

```sql
-- savepoint操作
-- 开启事务
begin;
-- 插入一条数据
insert into test values (8,'铃铛',55,11);
-- 添加一个保存点
savepoint ok1;
-- 再插入数据,比如出了一场
insert into test values (9,'大唐官府',66,22);
-- 回滚到之前的提交点
rollback to savepoint ok1;
-- 就可以开始重试操作，重试成功，commit，失败可以rollback;
commit;
```

# 十一、并发问题

## 11.1 事务的隔离级别

在不考虑隔离性的前提下，事务的并发可能会出现的问题：

* 脏读：读到了其他事务未提交的数据。（必须避免这种情况）
* 不可重复读：同一事务中，多次查询同一数据，结果不一致，因为其他事务修改造成的。（一些业务中这种不可重复读不是问题）
* 幻读：同一事务中，多次查询同一数据，因为其他事务对数据进行了增删吗，导致出现了一些问题。（一些业务中这种幻读不是问题）

针对这些并发问题，关系型数据库有一些事务的隔离级别，一般用4种。

* READ UNCOMMITTED：读未提交（啥用没用，并且PGSQL没有，提供了只是为了完整性）
* READ COMMITTED：读已提交，可以解决脏读（PGSQL默认隔离级别）
* REPEATABLE READ：可重复读，可以解决脏读和不可重复读（MySQL默认是这个隔离级别，PGSQL也提供了，但是设置为可重复读，效果还是串行化）
* SERIALIZABLE：串行化，啥都能解决（锁，效率慢）

PGSQL在老版本中，只有两个隔离级别，读已提交和串行化。在PGSQL中就不存在脏读问题。

## 11.2 MVCC

首先要清楚，为啥要有MVCC。

如果一个数据库，频繁的进行读写操作，为了保证安全，采用锁的机制。但是如果采用锁机制，如果一些事务在写数据，另外一个事务就无法读数据。会造成读写之间相互阻塞。 大多数的数据库都会采用一个机制 **多版本并发控制 MVCC** 来解决这个问题。

比如你要查询一行数据，但是这行数据正在被修改，事务还没提交，如果此时对这行数据加锁，会导致其他的读操作阻塞，需要等待。如果采用PostgreSQL，他的内部会针对这一行数据保存多个版本，如果数据正在被写入，包就保存之前的数据版本。让读操作去查询之前的版本，不需要阻塞。等写操作的事务提交了，读操作才能查看到最新的数据。 这几个及时可以确保  **读写操作没有冲突** ，这个就是MVCC的主要特点。

**写写操作，和MVCC没关系，那个就是加锁的方式！**

**Ps：这里的MVCC是基于 *读已提交* 去聊的，如果是串行化，那就读不到了。**

在操作之前，先了解一下PGSQL中，每张表都会自带两个字段

* xmin：给当前事务分配的数据版本。如果有其他事务做了写操作，并且提交事务了，就给xmin分配新的版本。
* xmax：当前事务没有存在新版本，xmax就是0。如果有其他事务做了写操作，未提交事务，将写操作的版本放到xmax中。提交事务后，xmax会分配到xmin中，然后xmax归0。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/557e8ed0b67c4ee4a849537d380c6aee.png)

基于上图的操作查看一波效果

事务A

```sql
-- 左，事务A
--1、开启事务
begin;
--2、查询某一行数据,  xmin = 630,xmax = 0
select xmin,xmax,* from test where id = 8;
--3、每次开启事务后，会分配一个事务ID 事务id=631
select txid_current();
--7、修改id为8的数据，然后在本事务中查询   xmin = 631, xmax = 0
update test set name = '铃铛' where id = 8;
select xmin,xmax,* from test where id = 8;
--9、提交事务
commit;
```

事务B

```sql
-- 右，事务B
--4、开启事务
begin;
--5、查询某一行数据,  xmin = 630,xmax = 0
select xmin,xmax,* from test where id = 8;
--6、每次开启事务后，会分配一个事务ID 事务id=632
select txid_current();
--8、事务A修改完，事务B再查询  xmin = 630  xmax = 631
select xmin,xmax,* from test where id = 8;
--10、事务A提交后，事务B再查询  xmin = 631  xmax = 0
select xmin,xmax,* from test where id = 8;
```

# 十二、锁

PostgreSQL中主要有两种锁，一个表锁一个行锁

PostgreSQL中也提供了页锁，咨询锁，But，这个不需要关注，他是为了锁的完整性

## 12.1 表锁

表锁显而易见，就是锁住整张表。表锁也分为很多中模式。

表锁的模式很多，其中最核心的两个：

* ACCESS SHARE：共享锁（读锁），读读操作不阻塞，但是不允许出现写操作并行
* ACCESS EXCLUSIVE：互斥锁（写锁），无论什么操作进来，都阻塞。

具体的可以查看官网文档：http://postgres.cn/docs/12/explicit-locking.html

表锁的实现：

先查看一波语法![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/c633685c9dee4d5c8b3f0be00082a827.png)

就是基于LOCK开启表锁，指定表的名字name，其次在MODE中指定锁的模式，NOWAIT可以指定是否在没有拿到锁时，一致等待。

```sql
-- 111号连接
-- 基于互斥锁，锁住test表
-- 先开启事务
begin;
-- 基于默认的ACCESS EXCLUSIVE锁住test表
lock test in ACCESS SHARE mode;
-- 操作
select * from test;
-- 提交事务，锁释放
commit;
```

当111号连接基于事务开启后，锁住当前表之后，如果使用默认的ACCESS EXCLUSIVE，其他连接操作表时，会直接阻塞住。

如果111号是基于ACCESS SHARE共享锁时，其他线程查询当前表是不会锁住得

## 12.2 行锁

PostgreSQL的行锁和MySQL的基本是一模一样的，基于select for update就可以指定行锁。

MySQL中有一个概念，for update时，如果select的查询没有命中索引，可能会锁表。

PostgerSQL有个特点，一般情况，在select的查询没有命中索引时，他不一定会锁表，依然会实现行锁。

PostgreSQL的行锁，就玩俩，一个for update，一个for share。
在开启事务之后，直接执行select * from table where 条件 for update;

```sql
-- 先开启事务
begin;
-- 基于for update 锁住id为3的数据
select * from test where id = 3 for update;
update test set name = 'v1' where id = 3;
-- 提交事务，锁释放
commit;
```

其他的连接要锁住当前行，会阻塞住。

# 十三、备份&恢复

防止数据丢失的第一道防线就是备份。数据丢失有的是硬件损坏，还有人为的误删之类的，也有BUG的原因导致误删数据。

正常备份和恢复，如果公司有DBA，一般咱们不用参与，BUT，学的Java，啥都得会点~~

在PostgreSQL中，有三种备份方式：

**SQL备份（逻辑备份）** ：其实就是利用数据库自带的类似dump的命令，或者是你用图形化界面执行导入导出时，底层就是基于这个dump命令实现的。备份出来一份sql文件，谁需要就复制给谁。

优点：简单，方便操作，有手就行，还挺可靠。

缺点：数据数据量比较大，这种方式巨慢，可能导出一天，都无法导出完所有数据。

**文件系统备份（物理备份）** ：其实就是找到当前数据库，数据文件在磁盘存储的位置，将数据文件直接复制一份或多份，存储在不同的物理机上，即便物理机爆炸一个，还有其他物理机。

优点：相比逻辑备份，恢复的速度快。

缺点：在备份数据时，可能数据还正在写入，一定程度上会丢失数据。 在恢复数据时，也需要注意数据库的版本和环境必须保持高度的一致。如果是线上正在运行的数据库，这种复制的方式无法在生产环境实现。

**如果说要做数据的迁移，这种方式还不错滴。**

**归档备份：（也属于物理备份）**

先了解几个概念，在PostgreSQL有多个子进程来辅助一些操作

* BgWriter进程：BgWriter是将内存中的数据写到磁盘中的一个辅助进程。当向数据库中执行写操作后，数据不会马上持久化到磁盘里。这个主要是为了提升性能。BgWriter会周期性的将内存中的数据写入到磁盘。但是这个周期时间，长了不行，短了也不行。
  * 如果快了，IO操作频繁，效率慢。
  * 如果慢了，有查询操作需要内存中的数据时，需要BgWriter现把数据从内存写到磁盘中，再提供给查询操作作为返回结果。会导致查询操作效率变低。
  * 考虑一个问题： **事务提交了，数据没落到磁盘，这时，服务器宕机了怎么办？**
* WalWriter进程：WAL就是write ahead log的缩写，说人话就是预写日志（redo log）。其实数据还在内存中时，其实已经写入到WAL日志中一份，这样一来，即便BgWriter进程没写入到磁盘中时，数据也不会存在丢失的问题。
  * WAL能单独做备份么？单独不行！
  * 但是WAL日志有个问题，这个日志会循环使用，WAL日志有大小的线程，只能保存指定时间的日志信息，如果超过了，会覆盖之前的日志。
* PgArch进程：WAL日志会循环使用，数据会丢失。没关系，还有一个归档的进程，会在切换wal日志前，将WAL日志备份出来。PostgreSQL也提供了一个全量备份的操作。可以根据WAL日志，选择一个事件点，进行恢复。

查看一波WAL日志：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/454b1e7dea28432c903d9441888d935a.png)

这些就是归档日志

> wal日志的名称，是三块内容组成，
>
> 没8个字符分成一组，用16进制标识的
>
> 00000001 00000000 0000000A
>
> 时间线         逻辑id       物理id

查询当前库用的是哪个wal日志

```sql
-- 查看当前使用的wal日志  查询到的lsn：0/47233270
select pg_current_wal_lsn();
-- 基于lsn查询具体的wal日志名称  000000010000000000000047
select pg_walfile_name('0/47233270');
```

归档默认不是开启的，需要手动开启归档操作，才能保证wal日志的完整性

修改postgresql.conf文件

```
# 开启wal日志的内容，注释去掉即可
wal_level = replica
fsync = on
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/4d17ad175581412ca336491fac8495a3.png)

```
# 开启归档操作
archive_mode = on
# 修改一小下命令，修改存放归档日志的路径
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/5837d178562f4bd1a7ae165ce0279617.png)

**修改完上述配置文件后，记得重启postgreSQL进程，才会生效！！！！**

归档操作执行时，需要保证/archive存在，并且postgres用户有权限进行w操作

构建/archive路径

```bash
# postgres没有权限在/目录下构建目录
# 切换到root，构建目录，将目录的拥有者更改为postgres
mkdir /archive
chown -R postgres. archive
```

在当前库中做大量写操作，接入到wal日志，重置切换wal日志，再查看归档情况

发现，将当前的正在使用的wal日志和最新的上一个wal日志归档过来了，但是之前的没归档，不要慌，后期备份时，会执行命令，这个命令会直接要求wal日志立即归档，然后最全量备份。

## 13.1 逻辑备份&恢复

PostgreSQL提供了pg_dump以及pg_dumpall的命令来实现逻辑备份。

这两命令差不多，看名字猜！

pg_dump这种备份，不会造成用户对数据的操作出现阻塞。

数据库不是很大的时候，pg_dump也不是不成！

查看一波命令：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/161b0a1f8e964025ba06ea0fef7c8a90.png)

这个命令从三块去看：http://postgres.cn/docs/12/app-pgdump.html

* 连接的信息，指定连接哪个库，用哪个用户~
* option的信息有就点多，查看官网。
* 备份的数据库！

操作一波。

备份老郑库中的全部数据。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/ffc85c98ffeb47e2807ce09713822089.png)

删除当前laozheng库中的表等信息，然后恢复数据

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/4a9ccc1f12db4835afa6ad121a499c01.png)

---

除此之外，也可以通过图形化界面备份，在库的位置点击备份就成，导出一个文本文件。

## 13.2 物理备份（归档+物理）

这里需要基于前面的文件系统的备份和归档备份实现最终的操作

单独使用文件系统的方式，不推荐毕竟数据会丢失。

这里直接上PostgreSQL提供的pg_basebackup命令来实现。

pg_basebackup会做两个事情、

* 会将内存中的脏数据落到磁盘中，然后将数据全部备份
* 会将wal日志直接做归档，然后将归档也备走。

查看一波pg_basebackup命令

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/d88fec3d4a9d4c68a7d0bdab95fb51cb.png)

先准备一个pg_basebackup的备份命令

```sh
# -D 指定备份文件的存储位置
# -Ft 备份文件打个包
# -Pv 输出备份的详细信息
# -U 用户名（要拥有备份的权限）
# -h ip地址  -p 端口号
# -R 复制写配置文件
pg_basebackup -D /pg_basebackup -Ft -Pv -Upostgres -h 192.168.11.32 -p 5432 -R
```

准备测试，走你~

* 提前准备出/pg_basebackup目录。记得将拥有者赋予postgres用户
  ```
  mkdir /pg_basebackup
  chown -R postgres. /pg_basebackup/
  ```
* 给postgres用户提供replication的权限，修改pg_hba.conf，记得重启生效![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/84c492f3d74e4ff1af40238f4419a500.png)
* 执行备份
  ```
  pg_basebackup -D /pg_basebackup -Ft -Pv -Upostgres -h 192.168.11.32 -p 5432 -R
  ```
* 需要输入postgres的密码，这里可以设置，重新备份。![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/ab3f9e3db3ad4f769499eaf50a29d806.png)
* 执行备份![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/232a538ba0604be5a7bbcb6f9649a728.png)![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/3bc6360d43664e8eb9fed78551507cff.png)

## 13.3 物理恢复（归档+物理）

模拟数据库崩盘，先停止postgresql服务，然后直接删掉data目录下的全部内容

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/3a608201df65426c879397aa42ca569a.png)

将之前备份的两个文件准备好，一个base.tar，一个pg_wal.tar

第一步：将base.tar中的内容，全部解压到 **12/data** 目录下

第二步：将pg_wal.tar中的内容，全部解压到 **/archive** 目录下

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/0aa71a31879044f3a91e18d59d2ada42.png)

第三步：在postgresql.auto.conf文件中，指定归档文件的存储位置，以及恢复数据的方式![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/af510e295f324f1fa2fd4f8daa21f997.png)

第四步：启动postgresql服务

```sql
systemctl start postgresql-12
```

第五步：启动后，发现查询没问题，但是执行写操作时，出错，不让写。需要执行一个函数，取消这种恢复数据后的状态，才允许正常的执行写操作。

```sql
select pg_wal_replay_resume();
```

## 13.4 物理备份&恢复（PITR-Point in time Recovery）

### 模拟场景

> 场景：每天凌晨02:00，开始做全备（PBK），到了第二天，如果有人14:00分将数据做了误删，希望将数据恢复到14:00分误删之前的状态？

1、恢复全备数据，使用PBK的全备数据恢复到凌晨02:00的数据。（数据会丢失很多）

2、归档恢复：备份中的归档，有02:00~14:00之间的额数据信息，可以基于归档日志将数据恢复到指定的事务id或者是指定时间点，从而实现数据的完整恢复。

### 准备场景和具体操作

1、构建一张t3表查询一些数据

```sql
-- 构建一张表
create table t3 (id int);
insert into t3 values (1);
insert into t3 values (11);
```

2、模拟凌晨2点开始做全备操作

```sh
pg_basebackup -D /pg_basebackup -Ft -Pv -Upostgres -h 192.168.11.32 -p 5432 -R
```

3、再次做一些写操作，然后误删数据

```sql
-- 凌晨2点已经全备完毕
-- 模拟第二天操作
insert into t3 values (111);
insert into t3 values (1111);
-- 误删操作  2023年3月20日20:13:26
delete from t3;
```

4、恢复数据（确认有归档日志）

将当前服务的数据全部干掉，按照之前的全备恢复的套路先走着

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/d88780d23c3f4242838d3eea6c940649.png)

然后将全备的内容中的base.tar扔data目录下，归档日志也扔到/archive位置。

5、查看归档日志，找到指定的事务id

查看归档日志，需要基于postgresql提供的一个命令

```
# 如果命令未找到，说明两种情况，要么没有这个可执行文件，要么是文件在，没设置环境变量
# 咱们这是后者
pg_waldump
# 也可以采用全路径的方式
/usr/pgsql-12/bin/pg_waldump
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/8f03f2d6f8504c14929ae94ea01c0cfa.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/7c8c722149394ba6b2164e6405f0ab92.png)

6、修改data目录下的恢复数据的方式

修改postgresql.auto.conf文件

将之前的最大恢复，更换为指定的事务id恢复

基于提供的配置例子，如何指定事务id

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/7eee8b46edbf4232866ef06cb99f45d5.png)

修改postgresql.auto.conf文件指定好事务ID

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/a9af4efd90a244289d829a22a83d234a.png)

7、启动postgreSQL服务，查看是否恢复到指定事务ID

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/3db16b30d57f472489b5294b42a56aad.png)

8、记得执行会后的函数，避免无法执行写操作

```
select pg_wal_replay_resume();
```

# 十四、数据迁移

PostgreSQL做数据迁移的插件非常多，可以从MySQL迁移到PostgreSQL也可以基于其他数据源迁移到PostgreSQL

这种迁移的插件很多，这里只说一个，pgloader（巨方便）

以MySQL数据迁移到PostgreSQL为例，分为几个操作：

1、准备MySQL服务（防火墙问题，远程连接问题，权限问题）

准备了一个sms_platform的库，里面大概有26W条左右的数据

2、准备PostgreSQL的服务（使用当前一直玩的PostgreSQL）

3、安装pgloader

pgloader可以安装在任何位置，比如安装在MySQL所在服务，或者PostgreSQL所在服务，再或者一个独立的服务都可以

我就在PostgreSQL所在服务安装

```bash
# 用root用户下载
yum -y install pgloader
```

4、准备pgloader需要的脚本文件

官方文档： https://pgloader.readthedocs.io/en/latest/

**记住，PostgreSQL的数据库需要提前构建好才可以！！！！**

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/49c3a1cda07d471faa6f512a678a60d5.png)

5、执行脚本，完成数据迁移

先确认pgloader命令可以使用

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/25e9ae8ccc184a32a8eef97e47d212b7.png)

执行脚本：

```
pgloader 刚刚写好的脚本文件
```

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/9a31fa692c33483589366fc36bbb9ab2.png)

# 十五、主从操作

PostgreSQL自身只支持简单的主从，没有主从自动切换，仿照类似Nginx的效果一样，采用keepalived的形式，在主节点宕机后，通过脚本的执行完成主从切换。

## 15.1 主从实现（异步流复制）

操作方式类似与之前的备份和恢复

### 1、准备环境：

| 角色    | IP            | 端口 |
| ------- | ------------- | ---- |
| Master  | 192.168.11.66 | 5432 |
| Standby | 192.168.11.67 | 5432 |

准备两台虚拟机，完成上述的环境准备

修改好ip，安装好postgresql服务

### 2、给主准备一些数据

```sql
create table t1 (id int);
insert into t1 values (111);
select * from t1;
```

### 3、配置主节点信息（主从都配置，因为后面会有主从切换的操作）

修改 **pg_hba.conf** 文件

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/8cef483420ca4233ae1043a4a343362f.png)

修改 **postgresql.conf** 文件

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/f0cd76ca47cf4fc1926655cc1bd53023.png)

提前构建好归档日志和备份目录，并且设置好拥有者

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/2746/1668770654044/99c3abbb9c98413ea16c2a80b5ed2928.png)

重启PostgreSQL服务

```
systemctl restart postgresql-12
```

### 4、从节点加入到主节点

关闭从节点服务

```
systemctl stop postgresql-12
```

删除从节点数据（删除data目录）

```
rm -rf ~/12/data/*
```

基于pbk去主节点备份数据

```sh
# 确认好备份的路径，还有主节点的ip
pg_basebackup -D /pgbasebackup -Ft -Pv -Upostgres -h 192.168.11.66 -p 5432 -R
```

恢复数据操作，解压tar包

```bash
cd /pgbasebackuo
tar -xf base.tar -C ~/12/data
tar -xf pg_wal.tar -C /archive
```

修改postgresql.auto.conf文件

```
# 确认有这两个配置，一般第一个需要手写，第二个会自动生成
restore_command = 'cp /archive/%f %p'
primary_conninfo = 'user=postgres password=postgres host=192.168.11.66 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
```

修改standby.signal文件，开启从节点备份模式

```
# 开启从节点备份
standby_mode = 'on'
```

启动从节点服务

```
systemctl restart postgresql-12
```

查看主从信息

* 查看从节点是否有t1表
* 主节点添加一行数据，从节点再查询，可以看到最新的数据
* 从节点无法完成写操作，他是只读模式
* 主节点查看从节点信息
  ```
  select * from pg_stat_replication
  ```
* 从节点查看主节点信息
  ```
  select * from pg_stat_wal_receiver
  ```

## 15.2 主从切换（不这么玩）

其实主从的本质就是从节点去主节点不停的备份新的数据。

配置文件的系统其实就是两个：

* standby.signal文件，这个是从节点开启备份
* postgresql.auto.conf文件，这个从节点指定主节点的地址信息

切换就是原主追加上述配置，原从删除上述配追

1、主从节点全部stop停止：………………

2、原从删除上述配置：…………

3、原从新主启动服务：………

4、原主新从去原从新主备份一次数据：pg_basebackup操作，同时做解压，然后修改postgresql.conf文件以及standby.signal配置文件

5、启动原主新从查看信息

## 15.3 主从故障切换

默认情况下，这里的主从备份是异步的，导致一个问题，如果主节点写入的数据还没有备份到从节点，主节点忽然宕机了，导致后面如果基于上述方式实现主从切换，数据可能丢失。

PGSQL在9.5版本后提供了一个pg_rewind的操作，基于归档日志帮咱们做一个比对，比对归档日志，是否有时间差冲突。

实现操作：

1、rewind需要开启一项配置才可以使用

修改postgresql.conf中的 **wal_log_hints = 'on'**

2、为了可以更方便的使用rewind，需要设置一下 **/usr/pgsql-12/bin/** 的环境变量

```
vi /etc/profile
  追加信息
  export PATH=/usr/pgsql-12/bin/:$PATH
source /etc/profile
```

3、模拟主库宕机，直接对主库关机

4、从节点切换为主节点

```
# 因为他会去找$PGDATA，我没配置，就基于-D指定一下PGSQL的data目录
pg_ctl promote -D ~/12/data/
```

5、将原主节点开机，执行命令，搞定归档日志的同步

* 启动虚拟机
* 停止PGSQL服务
  ```
  pg_ctl stop -D ~/12/data
  ```
* 基于pg_rewind加入到集群
  ```
  pg_rewind -D ~/12/data/ --source-server='host=192.168.11.66 user=postgres password=postgres'
  ```
* 如果上述命令失败，需要启动再关闭PGSQL，并且在执行，完成归档日志的同步
  ```
  pg_ctl start -D ~/12/data
  pg_ctl stop -D ~/12/data
  pg_rewind -D ~/12/data/ --source-server='host=192.168.11.66 user=postgres password=postgres'
  ```

6、修改新从节点的配置，然后启动

* 构建standby.signal
  ```
  standby_mode = 'on'
  ```
* 修改postgresql.auto.conf文件
  ```
  # 注意ip地址
  primary_conninfo = 'user=postgres password=postgres host=192.168.11.66 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
  restore_command = 'cp /archive/%f %p'
  ```
* 启动新的从节点
  ```
  pg_ctl start -D ~/12/data/
  ```
