【配置mysql复制环境】（一）基于bin-log的配置方式

> 基于mysql5.7

复制就是把一台MySQL服务（Master）上的数据拷贝到另外一台或几台MySQL服务器（slaves）上。复制默认是异步进行的 。 根据配置，可以复制MySQL 服务上的全部数据库、指定的数据库或是一个数据库中指定的几个表。

复制的好处：
- 一个横向扩展的方案： 读写分离
- 数据安全： salves 可以将复制处理挂起后做备份而不影响master
- 分析： 在线数据处理放在master上； 分析统计处理放到slaves上，不影响master的性能。
- 异地数据分布

复制的方式
- bin-log: 传统的方式
- GTIDs: 新方式

复制类型
- oneway asynchronous replication
- synchronous repalication

复制方式
- 基于语句的复制 Statement Based Replication (SBR) 
- 基于行的复制 Row Based Replication (RBR)
- 混合方式 Mixed Base Replication (MRB)

## 16.1 复制的配置
### 16.1.1 基于bin-log配置的概要
主库将更新事件写入bin-log。
从库读取主库的bin-log， 并执行bin-log的中的事件。
每个从库会获取完整的bin-log。 从库默认是执行bin-log中所有的事件。
但可以配置只执行特定数据库或特定数据表的事件。
不能说配置只执行某一个事件。

bin-log的坐标： 文件名和文件内的位置。

每一个主库和从库都要配置一个唯一的 server-id。 
从库上要配置主库的host name 和 bin-log 文件名、位置。


server-id: a unique ID which the master and each slave must be configured.
slave statement  : "CHANGE MASTER TO"

### 16.1.2 基于bin-log 的复制配置
一般步骤
- 在主库上打开bin-log，并配置唯一的server-id。 需要重启服务
- 为每个从库配置唯一的server-id。 需要重启服务
- 创建一个复制作业专用的的用户（可选）
- 创建数据快照前或启动复制处理前，需要记下主库的bin-log 坐标， 用来配置从库。
- 主库如果已经有了数据，需要将既有数据通过数据快照先拷贝到所有的从库上。 根据存储引擎的不同， 数据快照的方式也不同。 对于MyISAM ，需要停下所有的处理获取一个读锁，然后获取bin-log坐标并导出数据。如果这个过程种数据有更新，会造成数据快照与主库信息不匹配，最终得到的从库数据也是不正确的。 (如果是InnoDB， 则不需要加读锁， 事务可以充分保证数据快照的生成了)?。
- 配置从库连接主库：主库连接， 用户名密码， bin-log坐标

#### 16.1.2.1 主库配置
关闭MySQL 服务并修改 my.cnf 或 my.ini 文件。 
在[mysqld] 下面添加log-bin 和 server-id配置
server-id 的取值范围 [1 , (2的32次方)−1]
```
[mysqld]
log-bin=mysql-bin
server-id=1
```
注意：
- 没有配置server-id 的话， 主库会拒绝所有的从库连接请求
- 为了保证持久性和一致性， 使用InnoDB时要在主库的my.cnf文件中配置  innodb_flush_log_at_trx_commit=1 和 sync_binlog=1 in the master my.cnf file
- 确保主库没有打开skip-networking 配置， 否则从库连不上主库

#### 16.1.2.2 创建复制用户

```
mysql> CREATE USER 'repl'@'%.example.com' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%.example.com';
```

#### 16.1.2.3 获取bin-log坐标
第一步： 打开一个客户端session 执行 【FLUSH TABLES WITH READ LOCK】 并保持客户端session一直打开
```
mysql> FLUSH TABLES WITH READ LOCK;
```

第二步： 再打开一个客户端session 执行 【SHOW MASTER STATUS】 
```
mysql > SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 73       | test         | manual,mysql     |
+------------------+----------+--------------+------------------+
```
如果之前没有打开bin-log, 这里得到的file 和 position 都是空的， 这时从库配置filename 和 position 为 空字符串 ('') 和 4。

#### 16.1.2.4 生成数据快照

主库如果已经数据，需要将主库数据复制到每一个从库上。

- mysqldump：推荐， 尤其是InnoDB
- 复制数据文件： 更快速。 InnoDB不建议这么做

mysqldump

```
shell on master> mysqldump --all-databases --master-data > dbdump.db
```
--master-data： 配置会自动添加一个 CHANGE MASTER TO  语句， 这样从库就可以不用配置bin-log坐标了。

--all-databases： 所有的数据库
--ignore-table： 排除指定的表
--databases ： 指定数据库

#### 16.1.2.5 配置从库
前提： 
- 主库已经配置好
- bin-log 坐标，数据快照OK
- 主库的读锁已经释放
    + > mysql> UNLOCK TABLES;

在配置文件 [mysqld] 下配置：
    ```
    [mysqld]
    server-id=2
    ```
    一定要配置，不能和其它mysql服务冲突，否则连不上。
    从库不需要配置bin-log。 也可以配置bin-log，将这个从库当作其它mysql的主库。


没有既有数据需要导入：
- 重启服务
- 连接主库
```
    mysql> CHANGE MASTER TO
    ->     MASTER_HOST='master_host_name',
    ->     MASTER_USER='replication_user_name',
    ->     MASTER_PASSWORD='replication_password',
    ->     MASTER_LOG_FILE='recorded_log_file_name',
    ->     MASTER_LOG_POS=recorded_log_position;
```

如果有既有数据需要导入：
1. 启动从库时带上   --skip-slave-start  参数， 不启动复制处理
2. 导入dump文件（通过上面的配置导出的dump文件带有  CHANGE MASTER TO，后面就不用执行了 ） 
> shell> mysql < fulldb.dump

#### 16.1.2.6 复制环境中添加从库

我们可以在不影响主库的情况下向复制环境中添加一个新的从库。 直接拷贝现存从库的数据文件夹到新从库，并配置一个不同的 server-id (用户配置) 和 一个 server UUID (启动时生成)。

1. 关闭现存从库复制处理， 并记录复制状态。
```
mysql> STOP SLAVE;
mysql> SHOW SLAVE STATUS\G
```
2. 关闭现存从库
```
shell> mysqladmin shutdown
```
3. 将所有的数据文件从老的从库上拷贝到新从库， 
4. 拷贝完成后，将新从库数据文件中的 auto.cnf删除（重启后自动生成 server UUID）。
5. 重启现存从库
6. 为新的从库配置一个 server-id(唯一)
7. 启动新从库，带上 --skip-slave-start 参数。
8. 使用 【SHOW SLAVE STATUS 】 确认复制状态和之前记录的 源从库一致
9. 确认 server-id 和 server UUID 没有冲突
10. 【START SLAVE】

