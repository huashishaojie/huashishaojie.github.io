---
title: HDFS原理
date: 2018-05-28 20:02:28
categories: HDFS
tags: 
- HDFS
- hadoop
---
&emsp;随着互联网的发展，数据日益增多，增长超过了单机能够处理的上线，数据如何存储和处理成为了科技公司的难题，随着google的三篇论文的发布，大家终于找到了一个方案-分布式文件系统+MapReduce。Hadoop是参考google论文实现的，集成了分布式文件系统与分布式批处理平台。hadoop的设计目标是用来解决大文件海量存储和批处理的，为了避免单个节点故障导致数据丢失，设计副本冗余机制。 本文将主要分析一下几个方面：

 - HDFS的概念与架构
 - 读写流程分析
 - 使用场景与缺点

## HDFS的概念与架构
HDFS采用的master/slave架构。一个HDFS集群通常由一个Active的NameNode和若干DataNode组成，为了避免NameNode单点问题，通常会做一个NameNode的standby作为备份。在整个hdfs涉及到许多的核心概念，下面做一个简单介绍：

 - NameNode：
   NameNode是一个中心服务器，负责管理文件系统的名字空间以及客户端的访问，比如文件的打卡、关闭、重命名文件或者目录。它负责确定数据块到具体的存储节点的映射。在其同意调度下进行数据块的创建、删除、复制。
 - DataNode: DataNode是HDFS的实际存储节点，负责管理它所在节点的存储；客户端的读写请求。并且定期上报心跳和块的存储位置。
 - Block: HDFS上文件，从其内部看，一个文件其实是被分成一个或者多个数据块存储的，这些数据块存储在一组DataNode上。
 - Edits: 在HDFS发起的创建、删除等操作其实是一个事物，事物在NameNode上以Edit对象存储在edits文件中，持久化在NameNode的本地磁盘上。
 - FSimage: FSimage是NameNode的元数据存储快照，持久化在NameNode的本地磁盘上。

当NameNode重启的时候，NameNode从FSImage和Edits文件中读取数据，加载到内存中。

在HDFS体系来看，NameNode主要负责元数据的存储与操作，DataNode负责实际的存储。DataNode通常在一个机器上部署一个进程，这些机器分布式在多个机架上。整体架构如下图所示：
![HDFS架构](/images/hdfs-architecture.png)
## HDFS的读写流程
### 读流程分析
![HDFS读流程](/images/hdfs_read.png)
 - 客户端打开文件，通过rpc的方式向NameNode获取文件块的存储位置信息，NameNode会将文件中的各个块的所有副本DataNode全部返回，这些DataNode会按照与客户端的位置的距离排序。如果客户端就是在DataNode上，客户端可以直接从本地读取文件，跳过网络IO，性能更高。
 - 客户端调用read方法，存储了文件的前几个块的地址的DFSInputStream，就会连接存储了第一个块的最近的DataNode。然后通过DFSInputStream就通过重复调用read()方法，数据就从DataNode流动到客户端，当DataNode的最后一个块读取完成了，DFSInputSteam会关闭与DataNode的连接，然后寻找下一个快的最佳节点。这个过程读客户端来说透明的，在客户端那边来看们就像是只读取了一个连续不断的流。
 - 块是按顺序读的，通过 DFSInputStream 在 datanode 上打开新的连接去作为客户端读取的流。他也将会通过namenode
   来获取下一批所需要的块所在的 datanode 的位置(注意刚才说的只是从 namenode 获取前几个块的)。当客户端完成了读取，就在FSDataInputStream 上调用 close() 方法结束整个流程。

在这个设计中一个重要的方面就是客户端直接从 DataNode 上检索数据，并通过 NameNode 指导来得到每一个块的最佳    DataNode。这种设计允许 HDFS 扩展大量的并发客户端，因为数据传输只是集群上的所有 DataNode    展开的。期间，NameNode    仅仅只需要服务于获取块位置的请求（块位置信息是存放在内存中，所以效率很高）。如果不这样设计，随着客户端数据量的增长，数据服务就会很快成为一个瓶颈。 

### 写流程分析
![HDFS写流程](/images/hdfs_write.png)
 - 通过Client向远程的NameNode发送RPC请求；
 - 接收到请求后NameNode会首先判断对应的文件是否存在以及用户是否有对应的权限，成功则会为文件创建一个记录，否则会让客户端抛出异常；
 - 当客户端开始写入文件的时候，开发库会将文件切分成多个packets，并在内部以"data
   queue"的形式管理这些packets，并向Namenode申请新的blocks，获取用来存储replicas的合适的datanodes列表，列表的大小根据在Namenode中对replication的设置而定。
 - 开始以pipeline（管道）的形式将packet写入所有的replicas中。开发库把packet以流的方式写入第一个
   datanode，该datanode把该packet存储之后，再将其传递给在此pipeline中的下一个datanode，直到最后一个
   datanode， 这种写数据的方式呈流水线的形式。
 - 最后一个datanode成功存储之后会返回一个ack packet，在pipeline里传递至客户端，在客户端的开发库内部维护着 "ack
   queue"，成功收到datanode返回的ack packet后会从"ack queue"移除相应的packet。
 - 如果传输过程中，有某个datanode出现了故障，那么当前的pipeline会被关闭，出现故障的datanode会从当前的
   pipeline中移除，剩余的block会继续剩下的datanode中继续以pipeline的形式传输，同时Namenode会分配一个新的
   datanode，保持replicas设定的数量。

## HDFS的使用场景和缺点
### 使用场景
hdfs的设计一次写入，多次读取，支持修改。

大文件，在hdfs中一个块通常是64M、128M、256M，小文件会占用更多的元数据存储，增加文件数据块的寻址时间。

延时高，批处理。

高容错，多副本。

### 缺点
延迟比较高，不适合低延迟高吞吐率的场景

不适合小文件，小文件会占用NameNode的大量元数据存储内存，并且增加寻址时间

支持并发写入，一个文件只能有一个写入者

不支持随机修改，仅支持append

