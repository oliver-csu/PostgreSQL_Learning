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
