# PostgreSQL_Learning

## 一、PostgreSQL介绍

PostgreSQL是一个功能强大的 **开源** 的关系型数据库，底层基于C实现。

PostgreSQL的开源协议和Linux内核版本的开源协议是一样的。也就是BSD协议，这个协议基本和MIT开源协议一样，就是你可以对PostgreSQL进行一些封装，然后商业化来收费。

PostgreSQL的名字咋来的：之前叫Ingres，后面为了解决一些Ingres中的一些问题，作为后面的Ingres，就起名叫Postgre。

PostgreSQL版本迭代的速度比较快，现在最新的正式的发布版本，已经到了15.RELEASE。

PostgreSQL的版本选择一般有两种：

* 如果为了稳定的运行，推荐使用12.x版本。
* 如果想体验新特性，推荐使用14.x版本。

PostgreSQL允许跨版本升级，而且没有什么大问题。

PostgreSQL社区特别活跃，基本是三个月一发版，也就意味着很多常见的BUG都可以得到及时的修复。

PostgreSQL其实在国外使用的比较多，国内暂时还是以MySQL为主。

但是国内很多国产数据库都是基于PostgreSQL做的二次封装：比如华为GaussDB，还有腾讯的Tbase等。

甚至很多公司原来玩的Oracle，直接平转到PostgreSQL，同时国内的很多云厂商都支持PostgreSQL了。

PostgreSQL因为开源，所以有很多做数据迁移的工具，可以快速的从MySQL，SQLServer，Oracle直接平转到PostgreSQL，比如pgloader这样的数据迁移工具。

PostgreSQL的官方社区地址：https://www.postgresql.org/

PostgreSQL的国内社区地址：http://www.postgres.cn/v2/home

## 二、PostgreSQL和MySQL的区别

技术没有好坏之分，主要看一下是否符合业务的需要，能否解决业务需求；其次也要看社区的活跃度以及更新的频次。

MySQL不支持的几点内容：

* MySQL的数据类型不够丰富；
* MySQL不支持序列概念（Sequence）；
* 使用MySQL时，网上比较好用的插件；
* MySQL的性能优化监控工具不是很多，定位问题的成本是比较高；
* MySQL的主从复制没有一个官方的同步策略，同步问题难以解决；
* MySQL虽然开源但不够彻底。

PostgreSQL相对MySQL上述问题的特点：

* PostgreSQL的数据类型嘎嘎丰富；
* PostgreSQL是有序列的概念的；
* PostgreSQL的插件特别丰富；
* PostgreSQL支持主从复制的同步操作，可以实现数据的零丢失；
* PostgreSQL的MVCC实现和MySQL不大一样（PostgreSQL一行数据会存储多个版本，最多可以存储40亿个事务版本）。

## 三、PostgreSQL的安装

主要在Linux中安装，不推荐在Windows下安装。

Linux的版本尽量使用CentOS 7.x版本，最好是7.6或者是7.8版本。

去官网找安装的方式：

![官网安装方式](./assets/20241010/ce79cee2b6394311893c70e20da3a96a.png)

选择好PostgreSQL的版本，以及Linux的发行版本

![Linux安装12版本](./assets/20241010/a1bcec660b7e4beb937627a5c044f64b.png)

拿到命令直接在Linux中运行即可

```sh
# 下载PGSQL的rpm包
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
# 安装PGSQL12的软件程序，需要下载，需要等一会，一般不会失败，即便失败，他也会重新帮你找镜像
sudo yum install -y postgresql12-server
# 数据库初始化
sudo /usr/pgsql-12/bin/postgresql-12-setup initdb
# 设置开启启动项，并设置为开启自行启动
sudo systemctl enable postgresql-12
# 启动PGSQL
sudo systemctl start postgresql-12
```
这种属于Windows下的傻瓜式安装，基本不会出错。

如果出错，可能是如下的问题：

* 安装Linux的时候，一定要选择最小安装；
* Linux不能连接外网；
* Linux中的5432端口可能被占用。

PostgreSQL不推荐使用root管理，在安装成功postgreSQL后，他默认会给你创建一个用户：postgres

使用PostgreSQL前需要先切换到postgres

```
su postgres
```

切换到postgres用户后，直接输入psql即可进入到postgreSQL提供的客户端

```
# 进入命令行
psql
# 查看有哪些库，如果是新安装的，有三个库：postgres，template0，template1
```

Windows版本的下载地址：https://sbp.enterprisedb.com/getfile.jsp?fileid=1258242

## 四、PostgreSQL的配置

两个配置文件：

* PostgreSQL的远程连接配置；
* PostgreSQL的日志配置。

PostgreSQL的主要配置放在数据目录下的， **postgresql.conf** 以及 **pg_hba.conf** 配置文件

```sh
# 配置文件目录
ls -l /var/lib/pgsql/12/data
```

![配置文件](./assets/20241010/c3f3386b5adf4a92ad6c30d34313c8c1.png)

上图可以看到，PostgreSQL的核心文件，都属于postgres用户，操作的时候，别用root用户，容易出问题，先切换到postgres用户再操作。

### 远程连接配置

PostgreSQL默认情况下不支持远程连接的，这个跟MySQL一样。

* MySQL给mysql.user追加用户，一般是采用grant的命令添加；
* PostgreSQL要基于配置文件修改，才能制定用户是否可以远程连接。

修改pg_hba.conf配置文件

用户以及对应数据库和连接方式的编写模板：

![用户权限配置模板](./assets/20241010/df6c3f8af68240968efa999a51f6d9bd.png)

```
# 第一块
local：代表本地连接，host代表可以指定连接的ADDRESS
# 第二块
database编写数据库名，如果写all，代表所有库都可以连接
# 第三块
user编写连接的用户，可以写all，代表所有用户
# 第四块
address代表那些IP地址可以连接
# 第五块
method加密方式，这块不用过多关注，使用md5即可
# 示例配置：允许任意地址的全部用户连接所有数据库
host    all             all             0.0.0.0/0               md5
```

![pg_hba.conf示例配置](./assets/20241010/0db1d87dd92b4068901649bf73c61dcd.png)

为了实现远程连接，除了用户级别的这种配置，还要针对服务级别修改一个配置

服务级别的配置在postgresql.conf

![postgresql.conf原始配置](./assets/20241010/52be7e1942834f18bf7d3109b342b842.png)

发现默认情况下，PostgreSQL只允许localhost连接，直接配置为*即可解决问题

![postgresql.conf示例配置](./assets/20241010/6f72192536624fe6bd9ec40e411afbe3.png)

为了生效，需要重启服务：

```sh
# 使用root用户重启
sudo systemctl restart postgresql-12
```

### 数据库的日志配置

查看postgresql.conf文件

![postgresql.conf日志原始配置](./assets/20241010/210375d9a72e40639750216606d2e2a0.png)

PostgreSQL默认情况下，只保存7天的日志，循环覆盖。

```
# 代表日志是开启的。
logging_collector = on
# 日志存放的路径，默认放到当前目录下的log里
log_directory = 'log'
# 日志的文件名，默认是postgresql为前缀，星期作为后缀
log_filename = 'postgresql-%a.log'
# 默认一周过后，日志文件会被覆盖
log_truncate_on_rotation = on
# 一天一个日志文件
log_rotation_age = 1d
# 一个日志文件，没有大小限制
log_rotation_size = 0
```

# 五、PostgreSQL的基础操作

在psql命令行（客户端）下，执行了一次\l，查看所有的库信息

可以直接基于psql查看一些信息，也可以基于psql进入到命令行后，再做具体操作

![\l](./assets/20241010/dcd871238f4549649018fc8b07c6dc93.png)

可以直接基于psql操作。

```
可以使用psql --help，查看psql的命令
可以直接进入到命令行的原因是psql默认情况下，就是以postgres用户去连接本地的pgsql，所以可以直接进入
下面的图是默认的连接方式
```

![默认连接方式](./assets/20241010/490b99cc853d406e87a8f9b05a1c88e9.png)

后面都基于psql的命令行（客户端）去进行操作

命令不用背，需要使用的时候，直接找帮助文档，在psql命令行中，直接输入如下命令：

```
\help，即可查看到数据库级别的一些命令
\?，可以查看到服务级别的一些命令
```

## 5.1 用户相关

构建用户命令：

```sql
# create user默认有连接权限，而create role没有，但可以基于选项设置
CREATE USER 名称 [ [ WITH ] 选项 [ ... ] ]
CREATE ROLE 名称 [ [ WITH ] 选项 [ ... ] ]
```

创建一个超级管理员用户：

```sql
CREATE USER root WITH SUPERUSER PASSWORD 'root';
```

![创建超级管理员](./assets/20241010/b10afe02df1c45b6bdab8f03eced03f5.png)

退出psql命令行

![退出命令行](./assets/20241010/e31c67986b4d4570ad70200065d9e704.png)

尝试去用root用户登录的psql命令：

```sh
psql -h 192.168.11.32 -p 5432 -U root -W
```

根据提示发现，光有用户不让登录，用户需要有一个数据库，创建一个root库：

```
create database root;
```

![创建数据库](./assets/20241010/d338efe105a44718969fbb369450e38d.png)

可以在不退出psql的前提下，直接切换数据库

![切换数据库](./assets/20241010/248de5f3be7446f5be9adf5225560268.png)

也可以退出psql，重新基于psql命令去切换用户以及数据库

如果要修改用户信息，或者删除用户，可以查看

```sh
# 修改用户，直接基于ALTER命令操作
# 删除用户，直接基于DROP命令操作
```

如果要查看现在的全部用户信息

![查看全部用户信息](./assets/20241010/d85d11baa4474370b5a4c97a18c07581.png)

## 5.2 权限操作

权限操作前，要先掌握一下PGSQL的逻辑结构

![逻辑结构图](./assets/20241010/0ca8c4fb119f44d5befe8020a2cd416a.png)

PostgreSQL一个数据库中有多个schema，在每个schema下都有自己的相应的表信息，权限粒度比MySQL更细一些。

在PostgreSQL中，权限的管理分为很多多层：

* server、cluster、tablespace级别：基于pg_hba.conf去配置；
* database级别：通过grant命令操作；
* namespace、schema级别：使用不多，暂不了解；
* 对象级别：通过grant命令去设置。

对database或者是对象级别做权限控制可以直接基于grant命令去操作，查看grant命令的使用帮助：

```sh
# 查看grant命令
\help grant
```

## 小任务

* 构建一个用户
* 构建一个数据库
* 在这个数据库下构建一个schema（数据库默认有一个public的schema）
* 将这个schema的权限赋予用户
* 在这个schema下构建一个表
* 将表的select，update，insert权限赋予用户

## 完成上述操作

```sql
-- 准备用户
create user user_oliver with password 'password';
-- 准备数据库
create database database_oliver;
-- 切换数据库
\c db_oliver;
-- 构建schema
create schema schema_oliver;
-- 将schema的拥有者修改为oliver用户
alter schema schema_oliver owner to user_oliver;
-- 将database_oliver库下的schema_oliver的schema中的表的增，改，查权限赋予给user_oliver用户
grant select,insert,update on all tables in schema schema_oliver to user_oliver;
-- 用postgres用户先构建一张表
create table schema_oliver.test(id int);
-- 切换到laozheng用户。
\c user_oliver -password 
-- 报错：
-- 致命错误:  对用户"-user_oliver"的对等认证失败
-- Previous connection kept
-- 上述方式直接凉凉，原因是匹配连接方式时，基于pg_hba.conf文件去从上往下找
-- 找到的第一个是local，匹配上的。发现连接方式是peer。
-- peer代表用当前系统用户去连接PostgreSQL
-- 当前系统用户只有postgres，没有user_oliver，无法使用peer连接
-- 想构建user_oliver用户时，发现PostgreSQL的所有文件拥有者和所属组都是postgres，并且能操作的只有拥有者
```

![文件权限](./assets/20241010/7d9dc08d006c4b8fb7959b63f8138c07.png)

```
-- 基于上述问题，不采用本地连接即可。
-- 采用远程连接。
psql -h 127.0.0.1 -p 5432 -U user_oliver -W
-- 这样依赖，跳过了local链接方式的匹配，直接锁定到后面的host，host的连接方式是md5，md5其实就是密码加密了。
-- 登录后，直接输入
\dn
-- 查看到当前database下有两个schema
```

这种权限的赋予方式，可以用管理员用户去构建整体表结构，分配指定用户，赋予不同的权限，就不怕用户误操作了。
