﻿---
title: YARN工作原理及其工作流程
date: 2018-06-08 14:24:58
categories: YARN
tags: 
- YARN
- hadoop
---
![YARN工作原理](/images/yarn.png)
### 一 YARN组成元素及其功能
RM:YARN的管理者
NM:YARN的执行者

### 二 YARN工作原理
首先数据上传到集群中.然后将写好的程序打成架包通过命令提交MR作业.提交到集群后由集群管理者MR开始调度分配资源.到HDFS读取数据执行MapReduce相关进程对数据进行计算


## 三 具体流程
第一步: Client执行main()函数中runjob();开启作业

第二步:client向RM发送作业请求同时RM将作业id以及jar包存放路径返回给Client

第三步:Client会把Jar路径为前缀作业id为后缀作为唯一存放路径,将jar包写入到HDFS集群中,默认情况下jar包写10份,而其他数据只写3份 当改程序运行完后删除这些数据

第四步:Client再次将Jar存放地址(更为详细的描述)提交给RM.

第五步:RM将其放入调度器,向NM发送命令,NM开启MRAPPMaster进程,MR根据HDFS中jar包数据量为NM分配任务.

第六步:NM通过心跳机制接受调度器分配的任务

第七步:NM会开启内部YARNChild 

第八步:YARNChild根据命令到HDFS检索作业资源,

第九步:YARNChild开启MapTask 或者Reduce Task

第十步:map计算YARNChild调度的数据.
