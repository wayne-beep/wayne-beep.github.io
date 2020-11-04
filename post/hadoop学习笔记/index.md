


# 大数据Hadoop

## 1. HDFS

#### namenode

namenode管理文件系统的命名空间，维护着文件系统树以及整棵树内所有文件和目录。这些信息以两个文件的形式永久保存在本地磁盘上：命名空间镜像文件和 编辑日志文件。namenode也记录每个文件中各个块所在数据节点信息，但它并不永久保存块的位置信息，因为这些信息会在启动的时候，根据数据节点信息进行重建。

-  编辑日志：文件系统客户端执行写操作时（创建或移动文件），这些食物首先被记录到编辑日志中。热议时刻只有一个文件处于打开可写状态，在每个事物完成后，且在向客户端发送成功代码之前，文件都需要更新和同步。当namenode向多个目录写数据时，只有在所有写操作更新并同步到每个复本之后方可返回成功代码，以确保任何事物都不会因为机器故障而丢失。
- 镜像文件：每个镜像文件（fsimage）都是文件系统元数据的一个完整的永久性检查点。（前缀表示映像文件中的最后一个事物）。并非每个写操作都会更新该文件，因为fsimage是一个大型文件，频繁写会使系统运行极为缓慢。

因此在namenode发送故障，最近的fsimage文件将被载入到内存以重构元数据的最近状态，再从相关点开始向前执行编辑日志中记录的每个事务，事实上namenode启动阶段就是这样做的。在这个过程中，namenode运行在安全模式，意味着namenode的文件系统对于客户端来说是只读的。在安全模式下，各个datanode会向namenode发送最新的块列表信息，namenode了解到足够多的块位置信息之后，即可高效运行，如果namenode认为向其发送更新信息的datanode节点过少，则它会启动块复制过程，以将数据块复制到新的datanode节点。如果满足最小复本条件，namenode会在30s之后退出安全模式。所谓的最小副本条件是指在整个文件系统中有99.9%的块满足最小副本级别（默认是1，由dfs.namenode.replication.min属性设置）。查看是否处于安全模式：

```CODE
$ ./bin/hdfs dfsadmin -safemode get
Safe mode is OFF
```

如果namenode损坏，文件系统上所有的文件将会丢失，因为我们不知道如何根据datanode块重建文件。因此对于namenode的容错非常重要。hadoop对此提供了两种机制。

1. 备份那些组成文件系统元数据持久状态的文件。hadoop可以通过配置使namenode在多个文件系统上保存元数据的持久状态。这些写操作是实时同步的，而且是原子的。**一般的配置是将持久文件写入本地磁盘的同时，写入一个远程挂载的网络文件系统（NFS）。**
2. 另一种是运行一个辅助的namenode，但它不能用作namenode。这个辅助namenode的重要作用是定期合并编辑日志和命名空间镜像以防止编辑日志过大。这个辅助的namenode一般在另一台单独的物理机运行，因为它需要大量的CPU时间，并且需要与namenode一样的内存来执行合并操作。它会保留合并后的命名空间镜像副本，并且在namenode发生故障的时候启用。但是辅助namenode保存的状态总是滞后于主节点，所以在主节点全部失效时，会丢失部分数据。这种情况下，一般都是把NFS存储的元数据复制到辅助namenode上作为新的主namenode运行。

当namenode节点失效后启动一个新的namenode需要满足以下条件才能正常提供服务：

1. 将命名空间的镜像导入内存中

2. 重演编辑日志

3. 接收到足够多的来自datanode的数据块的报告并退出安全模式。

   对于一个大型的集群冷启动时间大概要30分钟甚至更长。

高可用实现：

HDFS HA通常由两个NameNode组成，一个处于Active状态，另一个处于Standby状态。

Active NameNode对外提供服务，而Standby NameNode则不对外提供服务，仅同步Active NameNode的状态，以便能够在它失败时快速进行切换。

Hadoop 2.0官方提供了两种HDFS HA的解决方案，一种是NFS，另一种是QJM。这里我们使用简单的QJM。在该方案中，主备NameNode之间通过一组JournalNode同步元数据信息，一条数据只要成功写入多数JournalNode即认为写入成功。通常配置奇数个JournalNode，这里还配置了一个Zookeeper集群，用于ZKFC故障转移，当Active NameNode挂掉了，会自动切换Standby NameNode为Active状态。

YARN的ResourceManager也存在单点故障问题，这个问题在hadoop-2.4.1得到了解决：有两个ResourceManager，一个是Active，一个是Standby，状态由Zookeeper进行协调。

YARN框架下的MapReduce可以开启JobHistoryServer来记录历史任务信息，否则只能查看当前正在执行的任务信息。

Zookeeper的作用是负责HDFS中NameNode主备节点的选举，和YARN框架下ResourceManaer主备节点的选举



#### datanode

datanode是文件系统的工作节点，它们根据需要存取检索数据库（受客户端和namenode调度），定期向namenode发送它们存储块的列表。

#### 客户端client

client通过namenode和datanode的交互来访问整个文件系统

#### 元数据机制与checkpoint

元数据存在namenode中，主要是描述数据属性的信息，用来支持如存储位置、历史数据、资源查找、文件记录功能。Namenode中的元数据就是记录hdfs存储信息数据的数据。

namenode主要负责三个功能，分别是(1)管理元数据 (2)维护目录树 (3)响应客户请求

hdfs的修改添加等信息先记录在edits.log。

如果存储在namenode节点的磁盘中，因为经常需要进行随机访问，还有响应客户请求，必然是效率过低。所以元数据metadata要存储在内存中！！！方便随机存储！

但如果只存在内存中，一旦断点，元数据丢失，整个集群就无法工作了！

因此解决办法是元数据存储在内存中，但在磁盘中有备份，这个备份就是fsImage，存放在namenode节点对应的磁盘中。

新的问题：当在内存中的元数据更新时，如果同时更新fsImage，就会导致效率过低，相当于第一种；但如果不更新，就会发生一致性问题，一旦namenode节点断点，就会产生数据丢失。

解决办法：引入edits.log文件(只进行追加操作，效率很高)。每当元数据有更新或者添加元数据时，修改内存中的元数据并追加到edits.log中。这样，一旦namenode节点断电，可以通过fsImage和edits.log的合并，合成元数据。

但是，如果长时间添加数据到edit.log中，会导致该文件数据过大，效率降低，而且一旦断电，恢复元数据需要的时间过长。因此，需要定期进行fsImage和edits.log的合并。

如果这个操作有namenode节点完成，又会效率过低。因此，引入一个新的节点secondaryNamenode，专门用于fsImage和edits.log的合并。

合并操作发生在secondary namenode中。

#### 具体的checkpoint执行过程如下：

1.secondary namenode请求主Namenode停止使用edits文件，暂时将新的写操作记录到一个新文件中，如edits.new。
 2.secondary namenode节点从主Namenode节点获取fsimage和edits文件（采用HTTP GET）
 3.secondary namenode将fsimage文件载入到内存，逐一执行edits文件中的操作，创建新的fsimage文件
 4.secondary namenode将新的fsimage文件发送回主Namenode（使用HTTP POST）
 5.主Namenode节点将从secondary namenode节点接收的fsimage文件替换旧的fsimage文件，用步骤1产生的edits.new文件替换旧的edits文件（即改名）。同时更新fstime文件来记录检查点执行的时间

#### 联邦HDFS

namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，这意味着，对一个拥有海量数据的超大集群来说，内存将会成为限制namenode横向扩展的瓶颈。2.x版本引入联邦HDFS允许添加namenode节点实现扩展。其中每个namenode管理文件系统命名空间中的一部分。例如一个namenode管理/data1下的所有文件，另一个namenode管理/data2下的所有文件

在联邦环境下，每个namenode维护一个命名空间卷（namespace volum），由命名空间的元数据和一个数据块池组成，数据块池包含该命名空间下文件的所有数据块。命名空间卷之间是相互独立的，两两之间并不通信，甚至一个namenode失效也不会影响由其他namenode维护的命名空间的可用性。数据块池不在进行切分，因此集群中的datanode需要注册到每个namenode，并且存储着来自多个数据块池中的数据块。

#### HDFS高可用

针对namenode失效引发的不可用问题，hadoop2增加的对hdfs高可用HA的支持。通过配置一对活动-备份（active-standby）namenode。当活动namenode失效，备用namenode就会接管他的任务并开始服务来自客户端的请求，不会有明显的中断。实现这一目标需要在架构上做如下修改：

- namenode之间通过高可用共享存储实现编辑日志的共享，当备用namenode接管后，它将通读共享编辑日志至末尾以实现活动namenode的状态同步，并继续读取由活动namenode写入的新条目。
- datanode需要同时向两个namenode发送数据块处理报告，因为数据块的映射信息存储在namenode的内存中而非磁盘。
- 客户端需要使用特定的机制来处理namenode失效的问题，这一机制对用户是透明的。
- 辅助namenode的角色被备用namenode所包含，备用namenode为活动namenode的命名空间设置周期性检查点

namenode默认使用zookeeper来确保仅有一个活动的namenode，每一个namenode实现了一个轻量级的故障转移控制器，其工作机制就是监视宿主namenode是否失效（通过简单的心跳机制实现），并且在namenode失效时进行故障切换。

## 2. 构建hadoop集群

### 2.1. 规划

#### master节点

- namenode
- 辅助namenode
- 资源管理器
- 历史服务器

十几台服务器规模可以将namenode和资源管理器放在一起，只要至少有一份元数据存储在远程文件系统中，随着集群规模增大完全可以分离它们。

#### 机架注意事项

对于多机架，描述清楚节点-机架的映射关系很有必要，这使得hadoop将MapReduce任务分配到节点时会倾向于机架内部传输。HDFS还能更加智能的放置副本，以取得性能和弹性的平衡。

### 2.2. 部署

#### 节点初始环境

- 操作系统优化

- 创建初始用户hadoop

  ```code
  
  ```

  

- 配置

#### 优化

- 磁盘：使用noatime挂载优化来减少最近访问文件的时间记录。通常情况下，yarn的本地磁盘地址与datanode的地址使用的相同的磁盘分区但是不同目录。
- 



## 管理Hadoop

### 日志审计

HDFS日志能够记录所有文件系统访问请求，有些组织需要这项特性来进行审计。对日志进行审计是log4j在INFO级别实现的。在默认配置下，次项特性并未启用，可以在hadoop-env.sh中增加以下命令实现：

```code
export HDFS_AUDIT_LOGGER="INFO,RFAAUDIT"
```

### 监控

使用jmx监控

### 日常维护

1. 元数据备份

   如果namenode的永久性元数据丢失或损坏，则整个文件系统将无法使用。可以在系统中分别保存若干份不同时间的备份（例如1小时，1天，一周）以保护元数据，方法1是直接保存这些元数据的复本，方法2是正和岛namenode上正在使用的文件中。最直接的元数据备份方法是使用dfsadmin命令下载namenode最新的fsimage文件的复本：

   ```code
   $ ./bin/hdfs dfsadmin -fetchImage /data/backup/fsimage.backup
   2020-10-04 21:56:55,414 INFO namenode.TransferFsImage: Opening connection to http://localhost:9870/imagetransfer?getimage=1&txid=latest
   2020-10-04 21:56:55,825 INFO common.Util: Combined time for file download and fsync to all disks took 0.01s. The file download took 0.01s at 200.00 KB/s. Synchronous (fsync) write to disk of /data/backup/fsimage.backup took 0.00s.
   
   $ ls  /data/backup/fsimage.backup
   /data/backup/fsimage.backup
   ```

   可以用脚本从准备存储fsimage存档文件的异地站点运行该命令。还需要测试复本的完整性，测试方法是启动一个本地namenode进程，查看它是否能够将fsimage和edits文件载入内存（可通过扫描namenode日志获得成功信息）

   

2. 数据备份

   备份级别？

3. 文件系统检查fsck

   定期（每天）在整个文件系统上运行HDFS的fsck（文件系统检查）工具，主动查找丢失或损坏的块

4. 文件均衡器

   定期运行均衡器工具，保持文件系统的datanode比较均衡，使不均衡的块儿分布重新平衡，避免部分datanode过于繁忙。
   
5. Datanode 进程启动时，启动DU线程定期执行du -sk命令统计各blockpool目录占用情况，会占用大量磁盘io，随着心跳汇报给namenode，执行周期默认600000ms，配置项为fs.du.interval。hadoop2.8.0;3.0.0-alpha1已经修复。解决方案：

   - 使用df命令替换du（df前提条件磁盘目录下存储只有hdfs单块池占用）
   - 增加检查时间随机抖动机制（将一个节点上同时产生的多个du操作，加个随机数，随机到集群的不同时间段，）

   1. 执行机制不同： Linux df和du执行原理机制的不同，du的数据是基于文件获取的，并非针对某个分区，执行时间受限于文件和目录个数；df直接使用 statfs系统调用，直接读取分区的超级块信息获取分区使用情况，针对整个分区，直接读取超级块，运行速度不受文件目录个数影响，执行很快。
   2. du和df不一致情况: 常见的df和du不一致情况就是文件删除的问题。当一个文件被删除后，在文件系统 目录中已经不可见了，所以du就不会再统计它了。然而如果此时还有运行的进程持有这个已经被删除了的文件的句柄，那么这个文件就不会真正在磁盘中被删除， 分区超级块中的信息也就不会更改。这样df仍旧会统计这个被删除了的文件。

### 委任和解除节点

通常情况下，节点同时运行datanode和节点管理，因而两者一般同时被委任和解除

1. 委任节点

   委任节点先配置hdfs-site.xml文件，指向namenode；其次配置yarn-site.xml文件，指向资源管理器，对于HDFS来说文件由dfs.hosts.include 属性设置，对于yarn来说文件由yarn.resourcemanager.nodes.include-path属性设置；最好启动datanode和资源管理器守护进程。namenode端需要配置dfs.hosts指定的文件内加入datanode网络地址。

2. 移除节点

   用户将拟退出的若干datanode告知namenode，Hadoop系统会将这些datanode停机之前将块复制到其他datanode。有了节点管理的支持，hadoop对故障容忍度更高。如果关闭一个正在运行的MapReduce任务的节点管理器，application master会检测到故障，并在其他节点上重新调度任务。

   接触旧节点的过程有exclude文件控制，对于HDFS来说文件由dfs.hosts.exclude 属性设置，对于yarn来说文件由yarn.resourcemanager.nodes.exclude-path属性设置

如果include文件为空，则所有节点都包含在include文件中，如果datanode同时出现在include和exclude文件中，则该节点可以连接，但很快就会被解除委任。

## hadoop相关开源项目

### 1. YARN

YARN（yet another resource negotiator）是hadoop的集群资源管理系统，hadoop2开始引入，最初的目的是为了改善MapReduce，由于有足够的通用性，同样可以支持其他的分布式计算模式。

YARN提供请求和使用集群资源的API，但这些API很少直接用于用户代码，相反，用户代码中用的是分布式计算框架提供的更高层的API，这些API建立在YARN之上且向用户隐藏了资源管理的细节。

#### 应用运行机制

YARN通过两类长期运行的守护进程提供自己的核心服务：管理集群上资源使用的**资源管理器（resource manager）**、运行在集群中**所有节点**上且能够启动核监控容器的**节点管理器（node manager）**。

容器：用于执行特定应用程序的进程，每个容器都有资源限制（CPU、内存等）。一个容器可以是Unix进程也可以是Linux Cgroup，取决于yarn的配置。

为了在YARN上运行一个应用，首先，客户端联系资源管理器，要求它运行一个application master进程。然后资源管理器找到一个能够在容器中启动application master的节点管理器，一个application master运行起来后能做什么依赖于应用本身：简单运行计算返回结果或向资源挂你器氢气更多的容器以用于运行一个分布式计算。后者是MapReduce YARN应用所做的事情。

#### 对比MapReduce

1. 可扩展性：YARN可以在更大规模集群上运行。MapReduce节点数达到4000，任务数达到40000时会遇到瓶颈，瓶颈来自于jobtracker必须同时管理作业和任务。而YARN里有资源管理器和application master的分离架构克服了这个局限。一个应用的每个实例都对应一个专门的application master
2. 可用性：当服务守护进程失败时，通过为另一个守护进程复制接管工作的状态以便继续服务，从而获得高可用性。然而jobtracker内存中大量快速变化的复杂状态是的改进jobtracker服务获得高可用性非常困难。由于YARN中jobtracker在资源管理器和application master之间进行了职责的划分，高可用的服务随之成为一个分而治之的问题：先为资源管理器提供高可用性，再为YARN应用提供高可用性。
3. 利用率：MapReduce每个tasktracker都配置有若干个固定长度的slot，都是静态分配的，在配置的时候就划分为map slot和reduce slot。一个slot仅能运行一个任务。YARN中，一个节点管理器管理一个资源池，而不是指定固定数目的slot。YARN上运行的MapReduce不会出现由于集群仅有map slot可用导致reduce任务必须等待的情况，如果能获得运行任务的资源，那么应用就会正常运行。
4. 多租户：YARN的最大优点在于向MapReduce以外的其他类型的分布式应用开放了hadoop。MapReduce仅仅是YARN应用中的一个。

#### 调度策略

- FIFO调度：维护队列，简单，但是不适合共享集群。达到应用会占用集群的所有资源。
- 容量调度器：一个独立的专门队列保证小作业一提交就可以启动，由于队列容量是为那个队列中的作业保留的，因此这种策略是以整个集群利用率为代价的，与FIFO调度器比，大作业执行的时间要长
- 公平调度器：不需要预留一定量的资源，因为调度器会在所有运行的作业之间动态平衡资源。第一个（大）作业启动时，它也是唯一运行的作业，因而获得集群中所有的资源。当第二个（小）作业启动时，它被分配到集群的一半资源，这样每个作业都能公平共享资源。在分配到一般资源时需要等待第一个作业释放资源，所以需要点时间。当小作业结束且不再申请资源后，大作业将回去再次使用全部的集群资源。最终的效果是即得到了较高的集群利用率，又能保证小作业能及时完成。CDH中默认使用公平调度。




### 2. Flume

设计Flume的宗旨是向hadoop批量导入基于事件的海量数据。一个典型的例子是里有Flume从一组web服务器中搜集日志文件，然后把这些文件中的日志事件转移到一个新的HDFS汇总文件以做进一步处理，其终点通常为HDFS，也可以写入HBase或者Solr。

### 3. HBase

HBase是一个在HDFS上开发的面向列的分布式数据库。如果需要实时的随机访问超大规模数据集，就可以使用HBase。HBase可以解决伸缩性的问题，它自底向上地进行构建，能够简单地通过增加节点来达到线性扩展。HBase并不是关系型数据库，它不支持SQL，但是它能做RDBMS（关系型数据库）不能做的事：在廉价硬件构成的集群上管理超大规模的稀疏表。

HBase的典型应用是webtable，一个以网页URL为主键的表，其中包含爬取的页面和页面的属性。

HBase描述为一个面向列的存储器，但更准确的说法它是一个面向列簇的存储器。物理上，所有的列簇成员都在一起存放在文件系统中。由于调优和存储都是在列簇这个层次上进行的，所以最好使所有列簇成员都有相同的**访问**模式（access pattern）和大小特征，比如对由于图像数据比较大（兆字节），元数据较小（Kb字节），因而他们分别存储在不同的列簇中。

简而言之：HBase和RDBMS中的表类似，只不过它的单元格有版本，行是排序的，儿只要列簇预先存在，客户随时可以把列添加到列簇中去。

#### 区域

HBase自动把表水平划分成区域（region）。每个区域由表中行的子集构成。每个区域由它所属于的表、它包含的第一行及其最后一行（不包括这行）来表示。一开始一个表只有一个区域，随着区域开始变大，等到它超出设定大小阈值，便会在某行的边界上把表分成两个大小基本相同的新分区。在第一次划分之前，所有加载的数据都放在原始区域所在的那台服务器（regionserver）。随着表变大，区域的个数会增加。**区域是在HBase集群上分布数据的最小单位**。用这种方式，一个因为太大而无法放在单台服务器上的表会被放到服务器集群上，其中每个节点都负责管理表所有区域的一个子集。表的加载也是使用这种方法把数据分布到各个节点。在线的所有区域按次序排列构成了表的所有内容。

#### 实现

正如HDFS和YARN是由客户端、从属机（slave）和协调主控机master（即HDFS的namenode和datanode，以及YARN的资源管理器和节点管理器）组成，HBase也采用相同的模型，它用一个master节点协调管理一个或多个regionserver从属机。HBase主控机master负责启动（bootstrap）一个全新的安装，把区域分配给注册的regionserver，恢复regionserver的故障。master负载很轻。regionserver负责0个或多个区域的管理以及相应客户端的读写请求。regionserver还负责区域的划分并通知HBase master有了新的子域。这样一来，主控机可以把父域设为离线，并用子域替换父域。

HBase依赖于zookeeper，默认情况下它管理一个zookeeper实例，可以通过配置使用已有的zookeeper集群。zookeeper集合体负责管理诸如hbase:meta目录的位置以及当前集群主控机地址等重要的信息。如果在区域的分配过程中有服务器崩溃，就可以通过zookeeper来进行分配协调。在zookeeper上管理分配事务的状态有助于在恢复时能够从崩溃服务器遗留的状态开始继续分配。在启动一个客户端到HBase集群的连接时，客户端必须至少拿到所传递的zookeeper集合体的位置。这样客户端才能访问zookeeper的层次结构，从而了解集群的属性（例如服务器的位置）。

类似于在hadoop中可以通过etc/hadoop/slaves文件中查看datanode和节点管理器一样，regionserver从属机节点列在HBase的conf/regionservers文件中。

HBase通过Hadoop文件系统API来持久化存储数据。多数人使用HDFS作为存储来运行HBase。但是，在默认情况下，除非另行指明，HBase会将存储写入本地系统，但是如果使用HBase集群，首要任务通常是把HBase的存储位置指向要使用的HDFS集群。

HBase内部保留名为hbase:meta的特殊目录表。他们维护着当前集群上所有区域的列表、状态和位置。hbase:meta表中的项使用区域名作为键，区域名由所属的表名、区域的起始行、区域的创建时间以及对整体进行MD5哈希值组成并且使用逗号分隔。区域发生变化时（即分裂、禁用|启用、删除、为负载均衡重新部署区域或由于regionserver崩溃儿重新部署区域）目录表会进行相应的更新。这样在集群上所有区域的状态嘻嘻就能保持最新的。

连接到zookeeper集群上的客户端首先查找hbase:meta的位置。然后客户端通过查找合适的hbase:meta区域来获取用户空间区域所在节点及其位置。接着、客户端就可以直接和管理那个区域的regionserver进行交互。客户端会缓存它们遍历hbase:meta时获取的信息，包括位置信息，用户空间区域的开始行和结束行，当访问发送错误时，即区域被移动了，客户端会再去查看hbase:meta获取区域的新位置。

到达regionserver的写操作首先追加到“提交日志“（commit log）中，然后加入内存中的memstore中。如果memstore满了，内容会被flush到文件系统。commitlog存放在HDFS中，因此即使一个regionserver崩溃，提交日志仍然可用。如果发现一个regionserver不能访问（通常因为服务器中的znode在zookeeper中过期），主控机器会根据区域对死掉的regionserver的提交日志进行分割，重新分配后，在打开并使用死掉的regionserver上的区域之前，这些区域会找到属于它们的从被分割提交日志中得到的文件，其中包含还没有被持久化存储的更新，这些更新会被replay以使区域恢复到服务器失败前的状态。（？不是很懂）

有一个后台进程负责在刷新文件个数到达一个阈值时压缩它们。它把多个文件重新写入一个文件，因为读操作检查的文件越少，它的执行效率就越高。在压缩时，进程会清理掉超出模式所设置最大的版本以及删除单元格或表示单元格为过期。在regionserver上另外有个一独立的进程监控着刷新文件的大小，一大文件大小超出预先设定的最大值，变对区域进行分割

#### HBase特性总结

- 没有真正的索引： 行是顺序存储的，每行中的列也是，所以不存在索引膨胀的问题，而且插入性能和表的大小无关
- 自动分区：  在表增长的时候，表会自动分裂成区域，并分布到可用的节点上
- 线性扩展和对于新节点的自动处理：  增加一个节点，把它指向现有的集群并运行regionserver，区域自动重新进行平衡，负载均衡分布。
- 普通商用硬件的支持：普通的商用机即可
- 容错：大量节点意味着每个节点的重要性不突出。不用担心单个节点失效。
- 批处理：MapReduce集成功能使我们可以用全并行的分布式作业根据”数据的位置“来处理

#### 高可用

同HDFS一样，HBase使用Zookeeper作为集群协调与管理系统，在HBase中其主要的功能与职责为：

- 存储整个集群HMaster与RegionServer的**运行状态**
- 实现HMaster的**故障恢复与自动切换**
- 为Client提供**元数据表**的存储信息

HMaster、RegionServer启动之后将会在Zookeeper上注册并创建节点（/hbasae/master 与 /hbase/rs/*），同时 Zookeeper 通过**Heartbeat的心跳机制来维护与监控节点状态**，一旦节点丢失心跳，则认为该节点宕机或者下线，将清除该节点在Zookeeper中的注册信息。

当Zookeeper中**任一RegionServer节点状态发生变化时，HMaster都会收到通知**，并作出相应处理，例如RegionServer宕机，HMaster重新分配Regions至其他RegionServer以保证集群整体可用性。

当HMaster宕机时（Zookeeper监测到心跳超时），**Zookeeper中的 /hbasae/master 节点将会消失，同时Zookeeper通知其他备用HMaster节点**，重新创建 /hbasae/master 并转化为active master。

#### 访问方式

客户端要访问HBase中的数据，只需要知道**Zookeeper集群的连接信息**，访问步骤如下：

- 客户端将从Zookeeper（/hbase/meta-region-server）**获得 hbase:meta 表存储在哪个RegionServer**，缓存该位置信息
- 查询该RegionServer上的 hbase:meta 表数据，查找要操作的 **rowkey所在的Region存储在哪个RegionServer中**，缓存该位置信息
- 在具体的RegionServer上**根据rowkey检索该Region数据**

可以看到，客户端操作数据过程并不需要HMaster的参与，通过Zookeeper间接访问RegionServer来操作数据。

#### HBASE的分裂（spilt）与压缩（Compaction）

HBASE的紧缩与分裂的发生都在Region中进行的，Region是HBASE中分布式存储和负载均衡的最小单元，不通的region可以分配在不通的RegionServer上。Region的数据存储依赖的是HDFS文件系统。概念回顾：

- **RegionServer**：管理和维护Master分配的Region，处理存储和查询数据请求
- **HLog**：写入数据时会先在HLog预写一份，以防RegionServer异常时，MemStore数据未刷入磁盘导致丢失
- **Region**：一张表为一个或多个region，region中保存着表中某段连续的数据
- **Store**：region中有一个或多个Store，一个Store对应一个列族
- **MemStore**：基于内存的存储单元（最新版本128M），写数据时先进入这里，当达到阈值大小或时间时，会触发flush到StoreFile（HFile）中，读数据优先在这里查询
- **StoreFile**：磁盘空间，用于存储MemStore刷过来的数据，以HFile文件存储数据。StoreFile数量的大小决定进行Compaction
- **HFile**：相当于StoreFile的一个包装,以Hadoop的二进制文件，StoreFile的底层文件，存储在HDFS上

**Region紧缩（Compaction）**Compaction是从一个Region的一个Store中选择部分HFile文件进行合并。合并原理是先从这些待合并文件中依次读出KeyValue数据，再由小到大排序后写入一个新的文件中，之后这个新生成的文件就会取代合并之前的文件对外提供服务。

- Minor_compation小缩

  指选取部分小的、相邻的HFile,将他们合并成一个更大的HFile，合并持续时间较短，合并过程中只是对文件进行合并而不会清理被标记删除或过期的数据

- Major_compation大缩

  指将一个Store中的所有HFile合并成一个HFile，在合并过程中会清理掉**被删除的数据，TTL过期数据及版本号超过设定版本号**的数据。Major_compaction持续时间比较长，整个过程会消耗大量的系统资源，对上层业务影响较大。因此，线上部分数据量较大的业务通常推荐关闭自动触发Major_compaction功能，改在业务相对空闲时手动触发。

  ```xml
  <!--低峰值执行major_compaction时间-->
  <property>
      <name>hbase.hregion.majorcompact.begintime</name>
      <value>00:00:00</value>
  </property>
  <property>
      <name>hbase.hregion.majorcompact.endtime</name>
      <value>01:30:00</value>
  </property>
  
  <!--关闭Major_compaction-->
  <property>
      <name>hbase.hregion.majorcompaction</name>
      <value>0</value>
   </property>
  ```

  

#### 常见问题

HBase使用HDFS的方式与MapReduce使用HDFS方式截然不同。在MapReduce中首先打开HDFS文件，然后map任务流式处理文件内容，最后关闭文件。在HBase中，数据文件在启动时就被打开，并在处理过程中始终保持打开状态，这是为了节省每次访问操作打开文件所需的代价。所以，HBase更容易遇到些MapReduce客户端碰不到的问题。

1. 文件描述符用完

   因为在连接的集群上一直保持文件的打开状态，所以一段时间后可能达到Hadoop设定的限制，会在日志中看到”too many open files"的错误信息。需要增加ulimit参数值解决

2. datanode线程用完

   Hadoop2同时运行的线程不能超过4096个，出现的线程用尽可能性很小
   
   

1. hadoop有几个配置文件 4个，两个最重要的core-site yarn-site ,mapreduce,hdfs-site
2. hbase问的多 ,zookeeper组网 ，bulk 内存设置多大，优化查询，升级
3. 无hive meta
4. storm说一下干啥的
5. kafka
6. spark 如何发jar包

#### Q&A

##### 1. 数据热点问题

出现热点问题原因

1. hbase的中的数据是按照字典序排序的，当大量连续的rowkey集中写在个别的region，各个region之间数据分布不均衡；

2. 创建表时没有提前预分区，创建的表默认只有一个region，大量的数据写入当前region；

3. 创建表已经提前预分区，但是设计的rowkey没有规律可循，设计的rowkey应该由regionNo+messageId组成

解决问题

1. 解决这个问题，关键是要设计出可以让数据分布均匀的**rowkey**

   ```code
   # 1 通过直接指定splits创建预分区
   create 'split01','cf1',SPLITS=>['1000000','2000000','3000000']
   # 2 通过指定文件创建预分区
   create 'split02','cf1',SPLITS_FILE=>'/data/hbase/split/split.txt'
   # split.txt内容：
   0001|
   0002|
   0003|
   # 3 java api create table并预分区
   ```

#### 优化

内存很小的时候，即使可以使用BucketCache来利用对外内存，但是使用对外内存的目的

内存超大的时候，GC对线上不会造成不稳定的影响，应尽量利用对外内存做读缓存，减小对内内存的压力，所以直接选择BucketCache

- **hbase.hregion.max.filesize** 默认10G，region


### 4. HIVE

原理：用户创建数据库、表信息，存储在hive的元数据库中；向表中加载数据，元数据记录hdfs文件路径与表之间的映射关系；执行查询语句，首先经过解析器、编译器、优化器、执行器，将指令翻译成MapReduce，提交到Yarn上执行，最后将执行返回的结果输出到用户交互接口。

Hive的设计目的是让精通SQL技能但Java编程技能相对弱的分析师能够对存放在HDFS中的大规模数据集进行查询。Hive的架构：

1. 用户接口主要有三个：CLI，Client 和 WUI。其中最常用的是CLI，Cli启动的时候，会同时启动一个Hive副本。Client是Hive的客户端，用户连接至Hive Server。在启动 Client模式的时候，需要指出Hive Server所在节点，并且在该节点启动Hive Server。 WUI是通过浏览器访问Hive
2. Hive将元数据存储在数据库中，如mysql、内嵌derby。Hive中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。
3. 解释器、编译器、优化器完成HQL查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在HDFS中，并在随后有MapReduce调用执行。
4. Hive的数据存储在HDFS中，大部分的查询、计算由MapReduce完成（包含*的查询，比如select * from tbl不会生成MapRedcue任务）。

#### 部署

按照Hive中metastore（元数据存储）不同位置分为三种方式:

1. 内嵌derby方式

   此方式是将元数据存在本地的Derby数据库，一般用于单元测试。缺点：使用derby存储方式时，运行hive会在当前目录生成一个derby文件和一个metastore_db目录。 这种存储方式的弊端是在同一个目录下同时只能有一个hive客户端能使用数据库，否则会出错误

2. Local方式

   这种存储方式需要在本地运行一个mysql服务器,一般运行在client中，需要将mysql的connector jar包拷贝到Hive所在目录的lib目录下

   ```CODE
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123' WITH GRANT OPTION;
   flush privileges;
   
   删除多余会对权限造成影响的数据
   刷新权限
   
   添加用户：
   CREATE USER 'hive'@'%' IDENTIFIED BY '123';
   授权用户：
   grant all privileges on hive_meta.* to hive@"%" identified by '123';
   flush privileges;
   ```

   虽然hive用户对hive_meta数据库是由操作权限的，但是这个数据库如果不存在，hive用户也是没有权限创建这个数据库，所以需要提前创建好hive_remote数据库;除此之外还要把证jline的jar包在yarn和hive中的版本要统一。

3. remote方式

   这种方式是基于存放元数据的数据库为mysql类型的方式上实现的，进一步实现了在多个节点上操作同一个hive数据库，实现远程连接，常用的一种方式。基于上一种方式，client节点上的配置不需要动，只需更改想要远程连接的节点上的配置文件；在客户端启动远程连接服务 `hive --service metastore` 即可。远程模式是metastore与hive服务分离，这样可以使数据库完全放在防火墙后面，有利于安全性。

- 先执行`cp hive-default.xml.template hive-site.xml`
- 搭建Hive之前要先将HDFS、Yarn以及MapRedecu的进程启动好，因为Hive的实习是在上述环境中进行的。
- 要注意Hadoop下的Yarn下lib中的jline这个jar包于Hive中的版本相统一，不然会出错。

### 4. Zookeeper

一是故障监控。每个NameNode将会和Zookeeper建立一个持久session，如果NameNode失效，那么此session将会过期失效，此后Zookeeper将会通知另一个Namenode，然后触发Failover；

二是NameNode选举。ZooKeeper提供了简单的机制来实现Acitve Node选举，如果当前Active失效，Standby将会获取一个特定的排他锁，那么获取锁的Node接下来将会成为Active。

**ZKFC：**

ZKFC是一个Zookeeper的客户端，它主要用来监测和管理NameNodes的状态，每个NameNode机器上都会运行一个ZKFC程序

**主要职责：**

一是健康监控。ZKFC间歇性的ping NameNode，得到NameNode返回状态，如果NameNode失效或者不健康，那么ZKFS将会标记其为不健康；

二是Zookeeper会话管理。当本地NaneNode运行良好时，ZKFC将会持有一个Zookeeper session，如果本地NameNode为Active，它同时也持有一个“排他锁”znode，如果session过期，那么次lock所对应的znode也将被删除；

三是选举。当集群中其中一个NameNode宕机，Zookeeper会自动将另一个激活。

## 部署CDH

合理的集群规划应该做到以下几点：

- 充分了解当前的数据现状
- 与业务方深入沟通，了解将会在集群上运行的业务，集群将会为业务提供什么服务
- 结合数据现状与业务，合理预估未来的数据量增长
- 盘点当前可用的硬件资源，包括机柜机架、服务器、交换机等
- 当前硬件资源不充足的情况下，根据数据评估情况作出采购建议
- 根据业务属性与组成，合理规划集群的部署架构
- 根据可用硬件资源，对集群节点的服务角色进行合理划分




## Q&A

1. 公司如何对namenode做高可用
2. 集群规模、数据量
3. namenode脑列的处理
4. 副本如何存储于不同的机架
5. 多租户容量管理
6. 集群日志管理，包括轮转，压缩和清理
7. datanode的目录们怎么指定
8. 集群做过的优化
9. 安全如何做的
10. 遇到过的问题
11. 没有理解辅助namenode机制
12. Hive用本地还是远程模式？

## 面试

### 1. Hadoop的各角色的职能？

Hadoop这头大 象奔跑起来，需要在集群中运行一系列后台(deamon）程序。不同的后台程序扮演不用的角色，这些角色由NameNode、DataNode、 Secondary NameNode、JobTracker、TaskTracker组成。其中NameNode、Secondary  NameNode、JobTracker运行在Master节点上，而在每个Slave节点上，部署一个DataNode和TaskTracker，以便这个Slave服务器运行的数据处理程序能尽可能直接处理本机的数据。对Master节点需要特别说明的是，在小集群中，Secondary  NameNode可以属于某个从节点；在大型集群中，NameNode和JobTracker被分别部署在两台服务器上。 

  Hadoop是一个能够对大量数据进行分布式处理的软件框架，实现了Google的MapReduce编程模型和框架，能够把应用程序分割成许多的小的工作单元，并把这些单元放到任何集群节点上执行。Hadoop提供的分布式文件系统（HDFS）主要负责各个节点的数据存储，并实现了 高吞吐率的数据读写。

  1、job 一个准备提交执行的应用程序称为“作业（job）”；

  2、task 从一个job划分出来、运行于各个计算节点的工作单元称为“任务（task）”。

  3、jobtracker 负责集群作业控制和资源管理，是mapreduce框架的大脑。

那么MapReduce在这个过程中到底做了那些事情呢？这就是本文以及接下来的一片博文将要讨论的问题，当然本文主要是围绕客户端在作业的提交过程中的工作来展开。先从全局来把握这个过程吧！

**JobClient**

  配置好作业之后，就可以向JobTracker提交该作业了，然后JobTracker才能安排适当的TaskTracker来完成该作业。（JobTracker是mapreduce框架中的主服务器，TaskTracker是从服务器）。

### 2. 集群角色

目前集群规模30台

- 类型：CPU40 mem176G 106TB 20台   CPU16 mem128G 14TB 12台
- 磁盘使用情况：1.9PB空间，使用220TB



多路多核，高主频CPU、大内存9

namenode节点100万文件的元数据要消耗800M内存，内存决定了集群保存文件数的总量，另外ResourceManager同时运行的作业会消耗一定的内存。



集群数据量确定集群规模：

​	一天增加10GB, 365天，原数据1TB，replacation=3， 1.5个mapreduce 计算完保存的数据，规划容量

　　（1TB + 10GB*365）*3*1.5 =20.53TB

#### 2.1. HDFS

- Namenode |Zkfc zkfailovercontroller  active 1 
- Namenode | Zkfc zkfailovercontroller standby  1 
- Datanode 24
- balancer 1  
- JournalNode 3
- zookeeper 5

#### 2.2. HBASE

- master active 1  与namenode在一起
- master backup | Thrift server| REST server 1  与namenode在一起，thirft为其他非java语言提供接口，rest提供web请求接口
- regionServer 30 和DataNode在一起

#### 2.3. HIVE 

都运行在master01和master02上

- hive metastore server 2
- Hiveserver 2
- webHCat server 1

#### 2.4. YARN 

- JobHistory server 1 与namenode在一起
- resourceManager active 与namenode在一起
- resourceManager standby 与namenode在一起
- Nodemanager 30 与HDFS在一起

#### 2.5. zookeeper

公共zk，5台

### 3. 面试题

1. 当前你们公司使用的Hadoop版本是什么

2. HDFS常见的数据压缩格式有哪些，介绍其中一种详细的实现方式

   - Gzip：压缩比在四种压缩方式中较高；hadoop本身支持，在应用中处理gzip格式的文件就和直接处理文本一样；有hadoop native库；大部分linux系统都自带gzip命令，使用方便。不支持切片
   - Bzip：支持split；具有很高的压缩率，比gzip压缩率都高；hadoop本身支持，但不支持native；在linux系统下自带bzip2命令，使用方便。压缩/解压速度慢；不支持native
   - Lzo：压缩/解压速度也比较快，合理的压缩率；支持split，是hadoop中最流行的压缩格式；支持hadoop native库；需要在linux系统下自行安装lzop命令，使用方便。hadoop本身不支持，需要安装；lzo虽然支持split，但需要对lzo文件建索引，否则hadoop也是会把lzo文件看成一个普通文件（为了支持split需要建索引，需要指定inputformat为lzo格式）。文件大于两百兆可以考虑，越大越有优势

   对大文件来说，不要使用不支持切分整个文件的压缩格式，因为会失去数据的本地特性，进而造成MapReduce应用效率低下

3. HDFS垃圾回收的时间模式是多久，如何修改该时间

4. HDFS如何生效机架感知，取消机架感知有什么问题

5. HDFS常见的运维操作有哪些，哪些操作是高危的，如果高危操作出现问题，如何解决

6. HDFS常见的故障是什么，如何处理，是否可以给出三种预案来防范大部分常见故障

7. 你经历过哪些严重的Hadoop故障

8. HDFS常用的IO压力测试工具有哪些

9. Hadoop哪些地方依赖于主机名，是否可以全部替换为IP呢（HDFS/YARN/SPARK）

10. HDFS有哪些核心的指标需要采集和监控，最重要的三个指标是什么

11. HDFS节点下线，如何提升其下线速度

12. HDFS常见的误删除数据场景，以及如何防止数据被误删除

13. HDFS集群对外提供的访问方式有几种，哪种最为常见，每种方式各自的优缺点和使用场景

14. HDFS你做过哪些性能调优，哪些是通用的，哪些是针对特定场景的

    - rack机架分配，指定服务器机架信息，有利于提高请求速率。机架存储策略为1.在本地机器hdfs目录下存储block，如果是客户端，则随机选择。2.在另外一个rack某个datanode上存储一个block。3.与第二个block同一个rack但不同机器上存储最后一个block
    - 禁用swap
    - 提升文件描述符数量
    - 挂载磁盘noatime来禁止更新文件访问时间
    - 关闭THP（transparent huge pages）内存大页自动化抽象层，hadoop作业时该参数会引起CPU偏高。
    - CPU内存比：1：4~8
    - Dfs.block.size: 134217728 数据块大小，默认128M
    - **dfs.namenode.handler.count** namenode同时和datanode通信的线程数，默认10。namenode工作线程池用来处理客户端的远程过程调用以及集群守护进程的调用。处理程序数量越多意味着更大的池来处理来自不同datanode的并发心跳以及并发元数据操作。设置方法为python -c 'import math ; print int(math.log(N) * 20)'  ，N为集群datanode节点的数量。配置过大会可能会导致内存溢出。
    - **dfs.datanode.handler.count  **datanode之间的并发线程量，,取决于系统的繁忙程度,设置太小会导致性能下降甚至报错。线程数的提高将增加DataNode的内存需求，因此，不宜过度调整这个数值。
    - **dfs.datanode.max.xcievers** datanode的最大连接数，如同Linux上文件句柄数
    - **dfs.datanode.balance.bandwidthPerse**执行start-balancer.sh的代码默认为1MB/s，将其调大，可以为20MB/s。
    - **dfs.replication**默认副本数，当许多任务同时读取一个文件时可能会造成瓶颈，可以通过只修改客户端的配置的方式增大来环境而且不影响全局
    - **dfs.datanode.max.transfer.threads**设置datanode在传输文件时最大线程数，通常设置为8192。如果集群中某台datanode主机上这个值比其他主机的大，会导致这台主机上存储数据比别的主机相对要多导致数据分布不均匀，及时balanace也不会均匀。
    - **io.file.buffer.size** hadoop缓冲区大小用于hdfs的文件的读写和map过程的中间结果输出，默认4K，可以增加到128K
    - **fs.trash.interval** 开启hdfs文件删除自动转移到垃圾箱的选项，值为垃圾箱文件清除时间，一般开启会可以防止误删，默认1440分钟（1天）。fs.trash.checkpoint.interval 垃圾回收的检查间隔，应小于或等于fs.trash.interval
    - **mapreduce.map.output.compress**是否对中间结果启用压缩，设置为true后，还可以设置mapreduce.map.output.compress.codec进行压缩算法选择，内置了snappy，还可以选择外装的lzo

15. Hbase调优

    - 设置regionserver的内存大小，默认1G，推荐设置为4G。修改conf/hbase-env.sh HBASE_HEAPSIZE=4g
    - **hbase.regionserver.handler.count** 修改客户端并发线程数，默认为10。设置规则为，当put和scans操作比较的多的时候，将其设置为比较小的值；当get和delete操作比较多的时候，将其设置为比较大的值。原因是防止频繁GC操作导致内存异常
    - 自定义hbase的分割和紧缩操作，默认情况下hbase的分割机制是当region大小达到hbase.hregion.max.filesize(10g)的时候进行自动分割，推荐每个regionserver的region个数在20~500个为最佳。hbase的紧缩机制是hbase的一个非常重要的管理机制，hbase的紧缩操作是非常消耗内存和cpu的，所以一般机器压力比较大的话，推荐将其关闭，改为手动控制
    - **hbase.balancer.period**  设置hbase的负载均衡时间，默认为300000(5分钟)，在负载比较高的集群上，将其值可以适当的改大。
    - **hfile.block.cache.size**修改hflie文件块在内存的占比，默认0.4。在读应用比较多的系统中，可以适当的增大该值，在写应用比较多的系统中，可以适当的减少该值，不过不推荐修改为0
    - 

16. Hadoop日常的运维操作有什么管理工具，已经搭建的集群如何使用ambari

17. Hadoop各类角色如何进行扩容，缩容，节点迁移（IP变更）

18. Hadoop各类角色的JVM参数配置如何设定

19. HDFS的block大小如何设置，取决于哪些因素

20. YARN的nodemanager上跑任务的时候，有时候会将磁盘全部打满，如何解决

21. HDFS集群多个业务方使用时如何提前做好运维规划，如权限，配额，流量突增，数据安全，目录结构

22. [HDFS中，小文件的定义是什么，如何对小文件进行统计分析，如何优化该问题](http://dongxicheng.org/mapreduce/hdfs-small-files-solution/)

23. HDFS的namenode如何进行主备切换

24. YARN的nodemanager导致机器死机，如何解决

25. 如何下线YARN的nodemanager节点，假如该节点持续在运行计算任务

26. YARN的nodemanager节点，从Active Nodes转为Lost Nodes，有哪些原因，在哪里设置

27. YARN的nodemanager节点如果转为Lost Nodes后，该节点上的计算任务是否还会正常继续

28. HDFS的快照原理简要介绍一下，为什么可以确保数据的安全性

29. YARN的yarn.nodemanager.local-dirs和yarn.nodemanager.log-dirs参数应该如何设置，有哪些常见的问题

30. distcp拷贝数据的时候，出现了java.lang.outofmemoryerror:java heap space，如何处理

31. 有两个hadoop集群，机器相同，磁盘占用相同，一个集群磁盘的使用率比较均匀，另一个集群磁盘使用率起伏较大（很多写满的，很多使用率很低的），那么第二个集群会有哪些问题

32. hdfs namenode启动慢，常见的原因有哪些？如何优化？

33. hadoop的hdfs、yarn配置的zookeeper，是否可以分开