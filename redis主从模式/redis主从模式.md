# redis主从模式

## 复制：
redis中，通过执行SLAVEOF命令或设置slaveof选项，让一个服务器（从服务器）去复制另一个服务器（主服务器）。

### 旧版本（2.8以前）复制功能
1. 同步（sync）：
<br/>将从服务器的数据库状态更新至主服务器当前所处的数据库状态。主要步骤：
    * 从服务器向主服务发送SYNC命令
    * 主服务器收到SYNC命令后，执行BGSAVE命令，在后台生成一个RDB文件，同时将此刻开始的所有写命令记录到一个缓冲区中。
    * 主服务器执行完BGSAVE命令后，将RDB文件发送给从服务器，从服务器解析RDB文件将状态恢复到主服务器执行BGSAVE时的数据库状态。
    * 主服务器将缓冲区内的写命令发送给从服务器，从服务器执行这些命令，使得主从数据库状态达到一致。

2. 命令传播（command propagate）：
<br/>当主服务器的数据库状态被修改（执行写命令），出现主从数据库状态不一致时，会将该条写命令发送（传播）给从服务器，从服务器执行改命令，让主从数据库状态重新达到一致。

3. 缺陷：
<br/>当复制过程中出现连接中断，重连之后，会重新执行复制过程（全量），不会增量复制，效率低下，耗费资源。

### 新版本（2.8以后）复制功能
<br/> 利用PSYNC命令代替SYNC命令来执行复制时的同步操作，有完整重同步和部分重同步2种模式。
1. 完整重同步：用于处理初次复制情况，与SYNC基本一致。

2. 部分重同步：用于处理断线后重复制的情况，增量复制。
<br/>部分重同步功能由以下三个部分构成:
    * 主服务器的**复制偏移量**（replication offset）和从服务器的复制偏移量
    * 主服务器的**复制积压缓冲区**（replication backlog）
    * **服务器的运行ID**（run ID）
    
#### 复制偏移量
执行复制的双方——主服务器和从服务器会分别维护一个复制偏移量：
* 主服务器每次向从服务器传播N个字节的数据时，就将自己的复制偏移量的值加上N
* 从服务器每次收到主服务器传播来的N个字节的数据时，就将自己的复制偏移量的值加上N

通过对比主从服务器的复制偏移量，程序可以很容易地知道主从服务器是否处于一致状态：
* 如果主从服务器处于一致状态，那么主从服务器两者的偏移量总是相同的
* 相反，如果主从服务器两者的偏移量并不相同，那么说明主从服务器并未处于一致状态
<br/><div align=center>![image](https://github.com/WangXing17/redisNote/blob/main/redis%E4%B8%BB%E4%BB%8E%E6%A8%A1%E5%BC%8F/img/%E5%A4%8D%E5%88%B6%E5%81%8F%E7%A7%BB%E9%87%8F.png)</div>
#### 复制积压缓冲区
复制积压缓冲区是由主服务器维护的一个固定长度（fixed-size）先进先出（FIFO）队列，默认大小为1MB。
当主服务器进行命令传播时，它不仅会将写命令发送给所有从服务器，还会将写命令入队到复制积压缓冲区里面，如下图所示。
<br/><div align=center>![image](https://github.com/WangXing17/redisNote/blob/main/redis%E4%B8%BB%E4%BB%8E%E6%A8%A1%E5%BC%8F/img/%E5%A4%8D%E5%88%B6%E7%A7%AF%E5%8E%8B%E5%8C%BA1.png)</div>

因此，主服务器的复制积压缓冲区里面会保存着一部分最近传播的写命令，并且复制积压缓冲区会为队列中的每个字节记录相应的复制偏移量，如下图所示。
<br/><div align=center>![image](https://github.com/WangXing17/redisNote/blob/main/redis%E4%B8%BB%E4%BB%8E%E6%A8%A1%E5%BC%8F/img/%E5%A4%8D%E5%88%B6%E7%A7%AF%E5%8E%8B%E5%8C%BA2.png)</div>

<br/>当从服务器重新连上主服务器时，从服务器会通过PSYNC命令将自己的复制偏移量offset发送给主服务器，主服务器会根据这个复制偏移量来决定对从服务器执行何种同步操作：
* 如果offset偏移量之后的数据（也即是偏移量offset+1开始的数据）仍然存在于复制积压缓冲区里面，那么主服务器将对从服务器执行部分重同步操作；
* 相反，如果offset偏移量之后的数据已经不存在于复制积压缓冲区，那么主服务器将对从服务器执行完整重同步操作

复制积压缓冲区建议设置大小：2*second*write_size_per_second
* second为从服务器断线后重新连接上主服务器所需的平均时间（以秒计算）
* write_size_per_second是主服务器平均每秒产生的写命令数据量（协议格式的写命令的长度总和）
#### 服务器的运行ID
除了复制偏移量和复制积压缓冲区之外，实现部分重同步还需要用到服务器运行ID（run ID）：
* 每个Redis服务器，不论主服务器还是从服务，都会有自己的运行ID；
* 运行ID在服务器启动时自动生成，由40个随机的十六进制字符组成，例如53b9b28df8042fdc9ab5e3fcbbbabff1d5dce2b3；

当从服务器对主服务器进行初次复制时，主服务器会将自己的运行ID传送给从服务器，而从服务器则会将这个运行ID保存起来。
当从服务器断线并重新连上一个主服务器时，从服务器将向当前连接的主服务器发送之前保存的运行ID：
* 如果从服务器保存的运行ID和当前连接的主服务器的运行ID相同，那么说明从服务器断线之前复制的就是当前连接的这个主服务器，主服务器可以继续尝试执行部分重同步操作；
* 相反地，如果从服务器保存的运行ID和当前连接的主服务器的运行ID并不相同，那么说明从服务器断线之前复制的主服务器并不是当前连接的这个主服务器，主服务器将对从服务器执行完整重同步操作。

## 主从模式优缺点：
优点：
* 支持主从复制，主机会自动将数据同步到从机，可以进行读写分离
* 为了分载 Master 的读操作压力，Slave 服务器可以为客户端提供只读操作的服务，写服务仍然必须由Master来完成
* Slave 同样可以接受其它 Slaves 的连接和同步请求，这样可以有效的分载 Master 的同步压力
* Master Server 是以非阻塞的方式为 Slaves 提供服务。所以在 Master-Slave 同步期间，客户端仍然可以提交查询或修改请求
* Slave Server 同样是以非阻塞的方式完成数据同步。在同步期间，如果有客户端提交查询请求，Redis则返回同步之前的数据

缺点：
* Redis不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的IP才能恢复（也就是要人工介入）
* 主机宕机，宕机前有部分数据未能及时同步到从机，切换IP后还会引入数据不一致的问题，降低了系统的可用性
* 如果多个 Slave 断线了，需要重启的时候，尽量不要在同一时间段进行重启。因为只要 Slave 启动，就会发送sync 请求和主机全量同步，当多个 Slave 重启的时候，可能会导致 Master IO 剧增从而宕机
* Redis 较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂
