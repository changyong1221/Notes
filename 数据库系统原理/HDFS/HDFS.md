# HDFS

## HDFS 分布式文件系统

### 文件系统的基本概述

- 文件系统定义：文件系统是一种存储和组织计算机数据的方法，它使得对其访问和查找变得容易。
- 文件名：在文件系统中，文件名是用于定位存储位置。
- 元数据（Metadata）：保存文件属性的数据，如文件名，文件长度，文件所属用户组，文件存储位置等。
- 数据块（Block）：存储文件的最小单元。对存储介质划分了固定的区域，使用时按这些区域分配使用。

### HDFS的概述

> HDFS(Hadoop Distributed File System)基于Google发布的GFS论文设计开发。HDFS是Hadoop技术框架中的分布式文件系统，对部署在多台独立物理机器上的文件进行管理。

![图片](pictures/640.png)

可用于多种场景，如：网站用户行为数据存储。生态系统数据存储。气象数据存储。

### HDFS的优点和缺点

其除具备其它分布式文件系统相同特性外，还有自己特有的特性：

- 高容错性：认为硬件总是不可靠的。
- 高吞吐量：为大量数据访问的应用提供高吞吐量支持。
- 大文件存储：支持存储TB-PB级别的数据。

#### 不适用场景

- 低时间延迟数据访问的应用，例如几十毫秒范围。

- - 原因：HDFS是为高数据吞吐量应用优化的，这样就会造成以高时间延迟为代价。

- 大量小文件 。

- - 原因：NameNode启动时，将文件系统的元数据加载到内存，因此文件系统所能存储的文件总数受限于NameNode内存容量。，那么需要的内存空间将是非常大的。

- 多用户写入，任意修改文件。

- - 原因：现在HDFS文件只有一个writer，而且写操作总是写在文件的末尾。

### 流式数据访问

- 流式数据访问：在数据集生成后，长时间在此数据集上进行各种分析。每次分析都将涉及该数据集的大部分数据甚至全部数据，因此读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要。
- 与流数据访问对应的是随机数据访问：它要求定位、查询或修改数据的延迟较小，比较适合于创建数据后再多次读写的情况，传统关系型数据库很符合这一点。

### HDFS的架构

HDFS架构包含三个部分：NameNode，DataNode，Client。

- NameNode：NameNode用于存储、生成文件系统的元数据。运行一个实例。
- DataNode：DataNode用于存储实际的数据，将自己管理的数据块上报给NameNode ，运行多个实例。
- Client：支持业务访问HDFS，从NameNode ,DataNode获取数据返回给业务。多个实例，和业务一起运行。

#### HDFS数据的写入流程

![图片](pictures/640-16637745560071.png)

- 业务应用调用HDFS Client提供的API，请求写入文件。
- HDFS Client联系NameNode，NameNode在元数据中创建文件节点。
- 业务应用调用write API写入文件。
- HDFS Client收到业务数据后，从NameNode获取到数据块编号、位置信息后，联系DataNode，并将需要写入数据的DataNode建立起流水线。完成后，客户端再通过自有协议写入数据到DataNode1，再由DataNode1复制到DataNode2, DataNode3。
- 写完的数据，将返回确认信息给HDFS Client。
- 所有数据确认完成后，业务调用HDFS Client关闭文件。
- 业务调用close, flush后HDFSClient联系NameNode，确认数据写完成，NameNode持久化元数据。

#### HDFS数据的读取流程

![图片](pictures/640-16637745560072.png)

- 业务应用调用HDFS Client提供的API打开文件。
- HDFS Client联系NameNode，获取到文件信息（数据块、DataNode位置信息）。
- 业务应用调用read API读取文件。
- HDFS Client根据从NameNode获取到的信息，联系DataNode，获取相应的数据块。(Client采用就近原则读取数据)。
- HDFS Client会与多个DataNode通讯获取数据块。
- 数据读取完成后，业务调用close关闭连接。

### HDFS的关键特性

![图片](pictures/640-16637745560073.png)

#### HDFS的高可靠性（HA）

- HDFS的高可靠性（HA）主要体现在利用zookeeper实现主备NameNode，以解决单点NameNode故障问题。
- ZooKeeper主要用来存储HA下的状态文件，主备信息。ZK个数建议3个及以上且为奇数个。
- NameNode主备模式，主提供服务，备同步主元数据并作为主的热备。
- ZKFC(ZooKeeper Failover Controller)用于监控NameNode节点的主备状态。
- JN(JournalNode)用于存储Active NameNode生成的Editlog。Standby NameNode加载JN上Editlog，同步元数据。

##### ZKFC控制NameNode主备仲裁

- ZKFC作为一个精简的仲裁代理，其利用zookeeper的分布式锁功能，实现主备仲裁，再通过命令通道，控制NameNode的主备状态。ZKFC与NN部署在一起，两者个数相同。

##### 元数据同步

- 主NameNode对外提供服务。生成的Editlog同时写入本地和JN，同时更新主NameNode内存中的元数据。
- 备NameNode监控到JN上Editlog变化时，加载Editlog进内存，生成新的与主NameNode一样的元数据。元数据同步完成。
- 主备的FSImage仍保存在各自的磁盘中，不发生交互。FSImage是内存中元数据定时写到本地磁盘的副本，也叫元数据镜像。

#### 元数据持久化

- EditLog:记录用户的操作日志，用以在FSImage的基础上生成新的文件系统镜像。
- FSImage:用以阶段性保存文件镜像。
- FSImage.ckpt:在内存中对fsimage文件和EditLog文件合并（merge）后产生新的fsimage，写到磁盘上，这个过程叫checkpoint.。备用NameNode加载完fsimage和EditLog文件后，会将merge后的结果同时写到本地磁盘和NFS。此时磁盘上有一份原始的fsimage文件和一份新生成的checkpoint文件：fsimage.ckpt. 而后将fsimage.ckpt改名为fsimage（覆盖原有的fsimage）。
- EditLog.new: NameNode每隔1小时或Editlog满64MB就触发合并,合并时,将数据传到Standby NameNode时,因数据读写不能同步进行,此时NameNode产生一个新的日志文件Editlog.new用来存放这段时间的操作日志。Standby NameNode合并成fsimage后回传给主NameNode替换掉原有fsimage,并将Editlog.new 命名为Editlog。

### HDFS联邦（Federation）

![图片](pictures/640-16637745560074.png)

- 产生原因：单Active NN的架构使得HDFS在集群扩展性和性能上都有潜在的问题，当集群大到一定程度后，NN进程使用的内存可能会达到上百G，NN成为了性能的瓶颈。
- 应用场景：超大规模文件存储。如互联网公司存储用户行为数据、电信历史数据、语音数据等超大规模数据存储。此时NameNode的内存不足以支撑如此庞大的集群。常用的估算公式为1G对应1百万个块，按缺省块大小计算的话，大概是128T (这个估算比例是有比较大的富裕的，其实，即使是每个文件只有一个块，所有元数据信息也不会有1KB/block)。
- Federation简单理解：各NameNode负责自己所属的目录。与Linux挂载磁盘到目录类似，此时每个NameNode只负责整个hdfs集群中部分目录。如NameNode1负责/database目录，那么在/database目录下的文件元数据都由NameNode1负责。各NameNode间元数据不共享，每个NameNode都有对应的standby。
- 块池（block pool）:属于某一命名空间(NS)的一组文件块。联邦环境下，每个namenode维护一个命名空间卷（namespace volume），包括命名空间的元数据和在该空间下的文件的所有数据块的块池。
- namenode之间是相互独立的，两两之间并不互相通信，一个失效也不会影响其他namenode。
- datanode向集群中所有namenode注册，为集群中的所有块池存储数据。
- NameSpace（NS）：命名空间。HDFS的命名空间包含目录、文件和块。可以理解为NameNode所属的逻辑目录。

### 数据副本机制

![图片](pictures/640-16637745560075.png)

#### 副本距离计算公式：

- Distance(Rack1/D1, Rack1/D1)=0，同一台服务器的距离为0。
- Distance(Rack1/D1, Rack1/D3)=2，同一机架不同的服务器距离为2。
- Distance(Rack1/D1, Rack2/D1)=4，不同机架的服务器距离为4。

#### 副本放置策略：

- 第一个副本在本节点。
- 第二个副本在远端机架的节点。
- 第三个副本看之前的两个副本是否在同一机架，如果是则选择其他机架，否则选择和第一个副本相同机架的不同节点，第四个及以上，随机选择副本存放位置。

如果写请求方所在机器是其中一个DataNode,则直接存放在本地,否则随机在集群中选择一个DataNode。

- Rack1：表示机架1。
- D1：表示DataNode节点1。
- B1：表示节点上的block块1。

### 配置HDFS数据存储策略

默认情况下，HDFS NameNode自动选择DataNode保存数据的副本。在实际业务中，存在以下场景：

- DataNode上存在的不同的存储设备，数据需要选择一个合适的存储设备分级存储数据。
- DataNode不同目录中的数据重要程度不同，数据需要根据目录标签选择一个合适的DataNode节点保存。
- DataNode集群使用了异构服务器，关键数据需要保存在具有高度可靠性的节点组中。

#### 配置HDFS数据存储策略--分级存储

##### 配置DataNode使用分级存储

![图片](pictures/640-16637745560076.png)

- HDFS的分级存储框架提供了RAM_DISK（内存盘）、DISK（机械硬盘）、ARCHIVE（高密度低成本存储介质）、SSD（固态硬盘）四种存储类型的存储设备。
- 通过对四种存储类型进行合理组合，即可形成适用于不同场景的存储策略。

#### 配置HDFS数据存储策略--标签存储

##### 配置DataNode使用标签存储：

![图片](pictures/640-16637745560077.png)

- 用户需要通过数据特征灵活配置HDFS文件数据块的存储节点。通过设置HDFS目 录/文件对应一个标签表达式，同时设置每个Datanode对应一个或多个标签，从而给文件的数据块存储指定了特定范围的Datanode。
- 当使用基于标签的数据块摆放策略，为指定的文件选择DataNode节点进行存放时，会根据文件的标签表达式选择出将要存放的Datanode节点范围，然后在这些Datanode节点范围内，选择出合适的存放节点。

支持用户将数据块的各个副本存放在指定具有不同标签的节点，如某个文件的数据块的2个副本放置在标签L1对应节点中，该数据块的其他副本放置在标签L2对应的节点中。支持选择节点失败情况下的策略，如随机从全部节点中选一个。简单的说：**给DataNode设置标签，被存储的数据也有标签。当存储数据时，数据就会存储到标签相同的DataNode中。**

#### 配置HDFS数据存储策略--节点组存储

##### 配置DataNode使用节点组存储：

![图片](pictures/640-16637745560088.png)

关键数据根据实际业务需要保存在具有高度可靠性的节点中，通过修改DataNode的存储策略，系统可以将数据强制保存在指定的节点组中。

**使用约束：**

1. 第一份副本将从强制机架组（机架组2）中选出，如果在强制机架组中没有可用节点，则写入失败。
2. 第二份副本将从本地客户端机器或机架组中的随机节点中（当客户端机器机架组不为强制机架组时）选出。
3. 第三份副本将从其他机架组中选出。
4. 各副本应存放在不同的机架组中。如果所需副本的数量大于可用的机架组数量，则会将多出的副本存放在随机机架组中。
5. 由于副本数量的增加或数据块受损导致再次备份时，如果有一份以上的副本缺失或无法存放至强制机架组，将不会进行再次备份。系统将会继续尝试进行重新备份，直至强制组中有正常节点恢复可用状态。
6. 简单的说：就是强制某些关键数据存储到指定服务器中。

### Colocation同分布

- 同分布(Colocation)的定义：将存在关联关系的数据或可能要进行关联操作的数据存储在相同的存储节点上。
- 按照下图存放，假设要将文件A和文件D进行关联操作，此时不可避免地要进行大量的数据搬迁，整个集群将由于数据传输占据大量网络带宽，严重影响大数据的处理速度与系统性能。

![图片](pictures/640-16637745560089.png)

- HDFS文件同分布的特性，将那些需进行关联操作的文件存放在相同数据节点上，在进行关联操作计算时避免了到其他的数据节点上获取数据，大大降低网络带宽的占用。
- 使用同分布特性，文件A、D进行join时，由于其对应的block都在相同节点，因此大大降低资源消耗。

![图片](pictures/640-166377455600810.png)

- Hadoop实现文件同分布，即存在相关联的多个文件的所有块都分布在同一存储节点上。文件级同分布实现文件的快速访问，避免了因数据搬迁带来的大量网络开销。

### HDFS数据完整性保障

HDFS主要目的是保证存储数据完整性，对于各组件的失效，做了可靠性处理。

- 重建失效数据盘的副本数据

- - DataNode向NameNode周期上报失败时，NameNode发起副本重建动作以恢复丢失副本。

- 集群数据均衡

- - HDFS架构设计了数据均衡机制，此机制保证数据在各个DataNode上分布是平均的。

- 元数据可靠性保证

- - 采用日志机制操作元数据，同时元数据存放在主备NameNode上。
  - 快照机制实现了文件系统常见的快照机制，保证数据误操作时，能及时恢复。

- 安全模式

- - HDFS提供独有安全模式机制，在数据节点故障，硬盘故障时，能防止故障扩散。

- 重建失效数据盘的副本数据

- - DataNode与NameNode之间通过心跳周期汇报数据状态，NameNode管理数据块是否上报完整，如果DataNode因硬盘损坏未上报数据块，
  - NameNode将发起副本重建动作以恢复丢失的副本。

- 安全模式防止故障扩散

- - 当节点硬盘故障时，进入安全模式，HDFS只支持访问元数据，此时HDFS 上的数据是只读的，其他的操作如创建、删除文件等操作都会导致失败。待硬盘问题解决、数据恢复后，再退出安全模式。

### HDFS架构其他关键设计要点说明

- 统一的文件系统

- - HDFS对外仅呈现一个统一的文件系统。

- 空间回收机制

- - 支持回收站机制，以及副本数的动态设置机制。

- 数据组织

- - 数据存储以数据块为单位，存储在操作系统的HDFS文件系统上。

- 访问方式

- - 提供JAVA API，HTTP方式，SHELL方式访问HDFS数据。

- 磁盘使用率

- - 比如磁盘100G，用了30G，使用率30%。

负载均衡避免了节点间数据分布不均匀，导致热点节点问题。

### 思考题

- HDFS是什么，适合于做什么？

- - 运行在通用硬件上的分布式文件系统。适合于大文件存储与访问、流式数据访问。

- HDFS包含那些角色？

- - NameNode、DataNode、Client。

- 请简述HDFS的读写流。

- - 读取：Client联系NameNode，获取文件信息。Client根据从NameNode获取到的信息，联系DataNode，获取相应的数据块；数据读取完成后，业务调用close关闭连接。
  - 写入：Client联系NameNode，NameNode在元数据中创建文件节点；Client联系DataNode并建立流水线，完成后，客户端再通过自有协议写入数据到 DataNode1，再由DataNode1复制到DataNode2,DataNode3；业务调用close关 闭连接；Client联系NameNode，确认数据写完成。



## HDFS概述

HDFS 是 Hadoop 的重要组成部分
HDFS 是 Hadoop Distribute File System 的简称，
意为：Hadoop 分布式文件系统。是 Hadoop 核心组件之一，作为最底层的分布式存储服务而存在。
分布式文件系统解决的问题就是大数据存储。

HDFS集群

![在这里插入图片描述](pictures/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ppdG9yWA==,size_16,color_FFFFFF,t_70.png)

管理者：NameNode
作用：负责管理，管理集群内各个节点。
负责管理整个文件系统的元数据（指的是数据的存放位置或存放路径）或名字空间
辅助管理者：SecondaryNameNode
作用：责辅助NameNode管理工作。
工作者：DataNode
作用：负责工作，进行读写数据。 周期向NameNode汇报。
负责管理用户的文件数据块(一个大的数据拆分成多个小的数据块)

Namenode作用
1、维护 管理文件系统的名字空间(元数据信息)
2、负责确定指定的文件块到具体的Datanode结点的映射关系。
3、维护管理 DataNode上报的心跳信息

DataNode作用
1、执行数据的读写（响应的是客户端）
2、周期性向NameNode做汇报（数据块的信息、校验和）
若datanode 10分钟没有向NameNode做汇报，表示已丢失（已宕机）
心跳周期 3秒 3、执行流水线的复制（一点一点复制）

HDFS 副本存放机制

![在这里插入图片描述](pictures/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1ppdG9yWA==,size_16,color_FFFFFF,t_70-166377471435623.png)


第一份数据来源于客户端
第二份存放的位置是与第一个副本在相同机架上，且不在同一个节点，按照一定的规则（cpu 内存 IO是用率，和硬盘剩余容量）找到一个节点存放
第三个副本的存放位置是与第一第二份数据副本不在同一个机架上，且逻辑与存放副本1和2的机架距离最近的机上按照一定的规则（cpu 内存 IO是用率，和硬盘剩余容量）找到一个节点进行存放
HDFS 特性
1、海量数据存储： HDFS可横向扩展，其存储的文件可以支持PB级别数据。

2、高容错性：节点丢失，系统依然可用，数据保存多个副本，副本丢失后自动恢复。 可构建在廉价（与小型机大型机比）的机器上，实现线性扩展(随着节点数量的增加，集群的存储能力，计算能力随之增加)。

3、大文件存储：DFS采用数据块的方式存储数据，将一个大文件切分成多个小文件，分布存储。

HDFS缺点：
1、 不能做到低延迟数据访问： HDFS 针对一次性读取大量数据继续了优化，牺牲了延迟性。

2、不适合大量的小文件存储 ：
A:由于namenode将文件系统的元数据存储在内存中,因此该文件系统所能存储的文件总数受限于namenode的内存容量。

B:每个文件、目录和数据块的存储信息大约占150字节。
由于以上两个原因，所以导致HDFS不适合大量的小文件存储

3、文件的修改； 不适合多次写入，一次读取（少量读取）

4、不支持多用户的并行写。
————————————————
版权声明：本文为CSDN博主「Zitor_」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/ZitorX/article/details/102915452





## HDFS漫画图解

### 1、系统构成和写数据过程

1.系统构成：客户端，NAMENODE（主节点），DATANODE（数据节点）

2.写数据原理

![img](pictures/SouthEast-166377476221726.jpeg)

![img](pictures/SouthEast-166377477829728.jpeg)

![img](pictures/SouthEast.jpeg)



### 2、读数据和容错

\1. 读数据原理

\2. 容错-故障类型和故障检测

![img](pictures/SouthEast-166377483569730.jpeg)

![img](pictures/SouthEast-166377484129732.jpeg)

![img](pictures/SouthEast-166377484784434.jpeg)

![img](pictures/SouthEast-166377485409336.jpeg)



### 3、容错和副本布局策略

整套漫画分为三篇，本文是第三篇，主要讲解了：

\1. 容错-读写容错

\2. 容错-DATANODE 故障

\3. 布局策略

\4. The End

![img](pictures/SouthEast-166377487616238.jpeg)

![img](pictures/SouthEast-166377488315840.jpeg)

![img](pictures/SouthEast-166377488874442.jpeg)

![img](pictures/SouthEast-166377489710944.jpeg)



## HDFS数据重分布Rebalance

Hadoop的HDFS集群在使用一段时间后，各个DataNode节点的磁盘使用率肯定会出现不平衡的情况，也就是数据量层面的数据倾斜，如图：

![HDFS的Block数据balancer重分布实战_数据](pictures/resize,m_fixed,w_750.webp)

引起这种情况的方式很多：

1.       添加新的Datanode节点

2.       人为干预将数据的副本数降低或者增加



我们都知道当HDFS出现数据不平衡的时候，就会造成MapReduce或Spark等应用程序无法很好的利用本地计算的优势，而且Datanode节点之间也没有更好的网络带宽利用率，某些Datanode节点的磁盘无法使用等等问题。



在Hadoop中，提供了hdfs balancer程序用来保证HDFS的数据平衡，我们先看一下这个程序的参数：

hdfs balancer –help

Usage: hdfs balancer

         [-policy <policy>]          the balancing policy: datanode or blockpool
    
         [-threshold <threshold>]       Percentage of disk capacity
    
         [-exclude [-f <hosts-file> | <comma-separated list of hosts>]]  Excludes the specified datanodes.
    
         [-include [-f <hosts-file> | <comma-separated list of hosts>]]  Includes only the specified datanodes.
    
         [-idleiterations <idleiterations>]    Number of consecutive idle iterations (-1 for Infinite) before exit.
    
         [-runDuringUpgrade]   Whether to run the balancer during an ongoing HDFS upgrade.This is usually not desired since it will not affect used space on over-utilized machines.



Generic options supported are

-conf <configuration file>     specify an application configuration file

-D <property=value>            use value for given property

-fs <local|namenode:port>      specify a namenode

-jt <local|resourcemanager:port>    specify a ResourceManager

-files <comma separated list of files>    specify comma separated files to be copied to the map reduce cluster

-libjars <comma separated list of jars>    specify comma separated jar files to include in the classpath.

-archives <comma separated list of archives>    specify comma separated archives to be unarchived on the compute machines.



The general command line syntax is

bin/hadoop command [genericOptions] [commandOptions]



选项的含义根据描述应该很好理解，其中-threshold参数是用来判断数据平衡的依据，值范围为0-100。默认值为10，表示HDFS达到平衡状态的磁盘使用率偏差值为10%，如果机器与机器之间磁盘使用率偏差小于10%，那么我们就认为HDFS集群已经达到了平衡的状态。



我们可以从CDH平台的CM上看到该参数是默认值和含义：

![HDFS的Block数据balancer重分布实战_.net_02](pictures/resize,m_fixed,w_750-166377502066447.webp)

该参数具体含义为：判断集群是否平衡的目标参数，每一个 Datanode 存储使用率和集群总存储使用率的差值都应该小于这个阀值，理论上，该参数设置的越小，整个集群就越平衡，但是在线上环境中，Hadoop集群在进行balance时，还在并发的进行数据的写入和删除，所以有可能无法到达设定的平衡参数值。



参数-policy表示的平衡策略，默认为DataNode。

该参数的具体含义为：应用于重新平衡 HDFS 存储的策略。默认DataNode策略平衡了 DataNode 级别的存储。这类似于之前发行版的平衡策略。BlockPool 策略平衡了块池级别和 DataNode 级别的存储。BlockPool 策略仅适用于 Federated HDFS 服务。



参数-exclude和-include是用来选择balancer时，可以指定哪几个DataNode之间重分布，也可以从HDFS集群中排除哪几个节点不需要重分布，比如：

hdfs balancer -include CDHD,CDHA,CDHM,CDHT,CDHO



除了上面的参数会影响HDFS数据重分布，还有如下的参数也会影响重分布，

dfs.datanode.balance.bandwidthPerSec, dfs.balance.bandwidthPerSec

该默认设置：1048576(1M/s)，个人建议如果机器的网卡和交换机的带宽有限，可以适当降低该速度，一般默认就可以了。

该参数含义如下：

HDFS平衡器检测集群中使用过度或者使用不足的DataNode，并在这些DataNode之间移动数据块来保证负载均衡。如果不对平衡操作进行带宽限制，那么它会很快就会抢占所有的网络资源，不会为Mapreduce作业或者数据输入预留资源。参数dfs.balance.bandwidthPerSec定义了每个DataNode平衡操作所允许的最大使用带宽，这个值的单位是byte，这是很不直观的，因为网络带宽一般都是用bit来描述的。因此，在设置的时候，要先计算好。DataNode使用这个参数来控制网络带宽的使用，但不幸的是，这个参数在守护进程启动的时候就读入，导致管理员没办法在平衡运行时来修改这个值，如果需要调整就要重启集群。





下面简单介绍一下balancer的原理：

![HDFS的Block数据balancer重分布实战_.net_03](pictures/resize,m_fixed,w_750-166377503318249.webp)

Rebalance程序作为一个独立的进程与NameNode进行分开执行。

步骤1：

Rebalance Server从NameNode中获取所有的DataNode情况：每一个DataNode磁盘使用情况。



步骤2：

Rebalance Server计算哪些机器需要将数据移动，哪些机器可以接受移动的数据。并且从NameNode中获取需要移动的数据分布情况。



步骤3：

Rebalance Server计算出来可以将哪一台机器的block移动到另一台机器中去。



步骤4,5,6：

需要移动block的机器将数据移动的目的机器上去，同时删除自己机器上的block数据。



步骤7：

Rebalance Server获取到本次数据移动的执行结果，并继续执行这个过程，一直没有数据可以移动或者HDFS集群以及达到了平衡的标准为止。



实战：

找一个比较空闲的的Datanode执行，建议不要在NameNode执行：

hdfs balancer -include CDHD,CDHA,CDHM,CDHT,CDHO

执行过程如下(部分)，大家可以对照上面的流程看日志，可能会更清楚一点：

16/07/11 09:35:12 INFO balancer.Balancer: namenodes  = [hdfs://CDHB:8022]

16/07/11 09:35:12 INFO balancer.Balancer: parameters = Balancer.Parameters [BalancingPolicy.Node, threshold = 10.0, max idle iteration = 5, number of nodes to be excluded = 0, number of nodes to be included = 5, run during upgrade = false]

Time Stamp               Iteration#  Bytes Already Moved  Bytes Left To Move  Bytes Being Moved

16/07/11 09:35:14 INFO net.NetworkTopology: Adding a new node: /default/192.168.1.130:50010

16/07/11 09:35:14 INFO net.NetworkTopology: Adding a new node: /default/192.168.1.131:50010

16/07/11 09:35:14 INFO net.NetworkTopology: Adding a new node: /default/192.168.1.135:50010

16/07/11 09:35:14 INFO net.NetworkTopology: Adding a new node: /default/192.168.1.138:50010

16/07/11 09:35:14 INFO net.NetworkTopology: Adding a new node: /default/192.168.1.139:50010

16/07/11 09:35:14 INFO balancer.Balancer: 2 over-utilized: [192.168.1.130:50010:DISK, 192.168.1.135:50010:DISK]

16/07/11 09:35:14 INFO balancer.Balancer: 1 underutilized: [192.168.1.131:50010:DISK]

16/07/11 09:35:14 INFO balancer.Balancer: Need to move 203.48 GB to make the cluster balanced.

16/07/11 09:35:14 INFO balancer.Balancer: Decided to move 10 GB bytes from 192.168.1.130:50010:DISK to 192.168.1.131:50010:DISK

16/07/11 09:35:14 INFO balancer.Balancer: Decided to move 10 GB bytes from 192.168.1.135:50010:DISK to 192.168.1.138:50010:DISK

16/07/11 09:35:14 INFO balancer.Balancer: Will move 20 GB in this iteration

16/07/11 09:36:00 INFO balancer.Dispatcher: Successfully moved blk_1074048042_307309 with size=134217728 from 192.168.1.130:50010:DISK to 192.168.1.131:50010:DISK through 192.168.1.130:50010

16/07/11 09:36:07 INFO balancer.Dispatcher: Successfully moved blk_1074049886_309153 with size=134217728 from 192.168.1.135:50010:DISK to 192.168.1.138:50010:DISK through 192.168.1.135:50010

16/07/11 09:36:09 INFO balancer.Dispatcher: Successfully moved blk_1074048046_307313 with size=134217728 from 192.168.1.130:50010:DISK to 192.168.1.131:50010:DISK through 192.168.1.130:50010

16/07/11 09:36:10 INFO balancer.Dispatcher: Successfully moved blk_1074049900_309167 with size=134217728 from 192.168.1.135:50010:DISK to 192.168.1.138:50010:DISK through 192.168.1.135:50010

16/07/11 09:36:16 INFO balancer.Dispatcher: Successfully moved blk_1074048061_307328 with size=134217728 from 192.168.1.130:50010:DISK to 192.168.1.131:50010:DISK through 192.168.1.130:50010

16/07/11 09:36:17 INFO balancer.Dispatcher: Successfully moved blk_1074049877_309144 with size=134217728 from 192.168.1.135:50010:DISK to 192.168.1.138:50010:DISK through 192.168.1.135:50010



如果你使用的是CDH集成平台，也可以通过CM来执行数据重分布：

步骤1：先选择HDFS组件的页面，如下：

![HDFS的Block数据balancer重分布实战_数据_04](pictures/resize,m_fixed,w_750-166377504475451.webp)

步骤2：找到页面右侧的操作选择，从下拉框中选择数据“重新平衡”选项

![HDFS的Block数据balancer重分布实战_hdfs_05](pictures/resize,m_fixed,w_750-166377505035353.webp)

步骤3：确定“重新平衡”就开始安装默认的设置规则重新分布DataNode的Block数据了，可以用CM的日志中查看具体的执行过程。

![HDFS的Block数据balancer重分布实战_hdfs_06](pictures/resize,m_fixed,w_750-166377505548855.webp)


-----------------------------------
©著作权归作者所有：来自51CTO博客作者香山上的麻雀的原创作品，请联系作者获取转载授权，否则将追究法律责任
HDFS的Block数据balancer重分布实战
https://blog.51cto.com/u_15278282/5155014
