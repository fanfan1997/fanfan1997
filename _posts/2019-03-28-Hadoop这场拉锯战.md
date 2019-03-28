---
layout: post
title: Hadoop这场拉锯战
tags: [大数据]
---

![官方图](https://user-gold-cdn.xitu.io/2019/3/27/169bdc06c4f302b4?w=884&h=182&f=png&s=138377)
## 概要
#### Hadoop是目前大数据领域最主流的一套分布式、高可用技术体系，包含了多种技术。包括HDFS（分布式文件系统），YARN（分布式资源调度系统），MapReduce（分布式计算系统），等等。稍后我们会逐个介绍。
#### 而且hadoop是基于Java开发的，所以在跨平台上有很大的优势，并且可以部署在廉价的计算机集群中。hadoop的核心是HDFS和MapReduce，HDFS是针对谷歌文件系统(GFS)的开源实现，具有较高的读写速度、容错性、可伸缩性，采用MapReduce的整合，可以在不了解分布式系统底层细节的情况下开发。这样就可以轻松的完成海量数据的存储和计算。这些特点都成为Hadoop有当前很高成就的必要条件。
#### Hadoop这个名字不是一个缩写，而是一个虚构的名字。该项目的创建者，Doug Cutting解释Hadoop的得名 ：“这个名字是我孩子给一个棕黄色的大象玩具命名的。我的命名标准就是简短，容易发音和拼写，没有太多的意义，并且不会被用于别处。小孩子恰恰是这方面的高手。” 哇哦! Best boy不愧是大数据的起源,导致之后所有和大数据有关的技术的吉祥物都和动物有关.
#### 既然说到Hadoop了，我们不得不说一下整个Hadoop的生态圈(有别于大数据生态圈),为什么叫Hadoop生态圈那?为什么不叫XXX那?那是因为人家是开山鼻祖。我想,一张图足矣。莫急,之后我们会一个一个的来。
![Hadoop生态圈](https://user-gold-cdn.xitu.io/2018/11/8/166f167726d1314e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
## HDFS
![](https://user-gold-cdn.xitu.io/2019/3/27/169be02a075f8bf8?w=1798&h=1216&f=png&s=231217)
#### HDFS全称是Hadoop Distributed File System，是Hadoop的分布式文件系统。它由很多机器组成，每台机器上运行一个DataNode进程，负责管理一部分数据。然后有一台机器上运行了NameNode进程，NameNode大致可以认为是负责管理整个HDFS集群的这么一个进程，他里面存储了HDFS集群的所有元数据。然后有很多台机器，每台机器存储一部分数据！以及相同文件的备份。(元数据:记录文件的路径)
#### 依托于结构图,我们可以这样理解，当客户端要上传一个文件的时候，先要读取到DataNodes的上下文,然后将上下文传给Namenode,然后NameNode会生成一个具体的位置，并且命令DataNodes开辟一块空间存放这个文件,因为是分布式文件系统,所以该文件会被复制好几份到其他的datanode中。这就是基本的工作流程。
#### Q:但是万一NameNode不小心宕机了可咋整？元数据不就全部丢失了？不慌,我们来看看HDFS优雅的解决方案。
#### A:每次内存里改完了，会生成一条editslog(元数据修改的操作日志)，然后写到磁盘文件里，不修改磁盘文件内容，就是顺序追加，这个性能就高多了。每次NameNode重启的时候，把editslog里的操作日志读到内存里回放一下，不就可以恢复元数据了？
#### Q:那随着上传文件的数量越来越多，那会不会导致每次重启之后读取的editslog磁盘文件岂不是很慢?
#### 我从网上找了一个图,感觉和官方说的八九不离十。
![](https://user-gold-cdn.xitu.io/2018/11/13/1670dc062e2fa9e8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
#### A:没有事情,Hadoop考虑到这个问题了。引入一个新的磁盘文件叫做fsimage，然后再引入一个JournalNodes集群以及一个Standby NameNode（备节点）每次ActiveNameNode(主节点)在写入一个新的元数据之后都会生成一条新的editslog，除了要写入磁盘文件，还要写入JournalNodes集群中，然后Standby NameNode就可以从JournalNodes集群拉取edits log，应用到自己内存的文件目录树里，跟Active NameNode保持一致。而且Standby NameNode会定时的向磁盘文件fsimage中写入整个文件目录树的元数据。这个操作就是所谓的checkpoint检查点操作。然后会清空Active NameNode的edits log文件，这个文件可能已经有几万行了。然后Active NameNode继续写edits log，在写了几十行的时候重启了，Active NameNode会把fsimage中的元数据全部读过来，然后再读edits log磁盘文件中的元数据。这样就能够保证元数据不丢失。在上图中我们可以看到有两个NameNode一个对外提供服务(Active NameNode)，另外一个就是纯接受edits log以及执行定期的checkpoint的备用节点(Standby NameNode)如此的话，如果主挂了，那么备用节点也可以上的。因为他们的元数据结构几乎是一模一样的。这就是HDFS的主备高可用故障迁移机制嘛!
## YARN
![](https://user-gold-cdn.xitu.io/2019/3/27/169be40b91bc72e4?w=1100&h=790&f=png&s=269157)
#### yarn的基本思想其实就是要将资源管理和作业调度分解为单独的守护进程,2.0之后既然要将1.0时候的资源调度分开,那就需要一个全局的ResourceManager以及每个应用程序上都要有一个ApplicationMaster。ResourceManager和NodeManager构成了数据计算框架。
#### ResourceManager拥有在系统中的所有应用程序之间仲裁资源的最终权限,他有两个主要组件：Scheduler和ApplicationsManager(Scheduler负责调度资源ApplicationsManager负责接收各NodeManager的情况以和Scheduler配合)。NodeManager是每台机器的代理，负责容器(Container)，监视其资源使用情况（CPU，内存，磁盘，网络）并将其报告给ResourceManager中的ApplicationsManager。
#### 而每个应用程序运行在Container中并且ApplicationMaster实际上是一个特定于框架的库，其任务是协调来自ResourceManager的资源，执行MR,并且跟踪 Job 的运行进度，并收集 task 的进度和完成情况并且提交
#### 以上内容其实特别抽象，我们举个栗子:我们将ResourceManager实例化为一个公司，要想让公司做的更好，那么需要底下的人多多谏言,那么就需要有收集谏言的部门和具体的执行部门，Container就好比是公司的服务器,那么服务器里边跑的是应用程序(MR Application),公司只有通过职员A(Application Master)才能知道应用程序的运行结果,也就是说职员A需要将应用程序的结果/运行情况告知给公司(Container中MR程序的task进度需要通过Application Master同步给ResourceManager)。但是这个时候系统集成人员突然发现有几台服务器的内存条不够了。那么就需要将这个需求提供给收集部门(ApplicationManager)那么收集部门就需要将建议提交给执行部门(Scheduler)。然后执行部门才会通过研发部系统集成人员将内存条安装到服务器上。然后职员A才能在服务器上运行程序。如此道理。其实在我们的生活当中这就是一个很常见的例子,要不然说万物归于自然。生活中我们还得多双发现的眼睛啊。
## MapReduce
![](https://user-gold-cdn.xitu.io/2018/9/18/165ea8b5dc9b0500?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
#### 这张图虽然不是官方的,但是也是能够完美的描述_MR共有四个部分.包括(Job、Map、Shuffle、Reduce)
#### 另外,官方是这样描述他的
![](https://user-gold-cdn.xitu.io/2019/3/28/169c209607496467?w=3396&h=642&f=png&s=312646)
#### 大致意思其实就是:HadoopMapReduce是一个软件框架，用于轻松(别听他瞎扯啊!)编写应用程序，以可靠，容错的方式在大型集群（数千个节点）的商用硬件上并行处理大量数据（多TB数据集）MapReduce作业通常将输入数据集拆分为独立的块，这些块由map任务以完全并行的方式处理。在框架的Shuffle中对map的输出进行排序，然后输入到reduce任务。通常，作业的输入和输出都存储在文件系统中。该框架负责调度任务，监视任务并重新执行失败的任务。通常，计算节点和存储节点是相同的，即MapReduce框架和Hadoop分布式文件系统在同一组节点上运行然后，Hadoop任务客户端将任务（jar /可执行文件等）和配置提交给ResourceManager，然后ResourceManager负责将相关的资源分配给容器，安排任务并监视它们，NodeManager为任务提供状态和诊断信息。
#### 其实大家可以发现,MR真正需要了解的东西很少，真正的还是在YARN资源调度这一块。MR程序不过只是运行在Container中的一个小部分。真正了解YARN对你的帮助是很大的。
#### 唯一需要了解的还是MR的四部分,包括map在输出之前是如何进行拆分的,以及输出了之后将文件落地到了HDFS上Shuffle是如何进行排序的，以及Shuffle对于排序有什么更好的方法吗？以及Shuffle排序完成输出到Reduce中的时候，Reduce是如何工作的。
#### easy!只需要了解这些，其余的我们将MRApp拉倒服务器上跑就行了。官网上都有[示例](http://hadoop.apache.org/docs/r3.1.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html)可以自己建个java项目,或者Maven拉一下jar包，做一下操作,很简单的。因为现在如果不是EB级别的数据要分析的话,是没有人用MR的,现在最好的分析框架是Spark，所以，我们对MR的介绍就这么多。大家只需要会写简单的程序了解我上边所说到的内容就即可。无需再多。
### The End 既然上边说到如果不是EB级别的数据的话是不用MR的,为什么那?这个时候我们就有必要先来铺垫一下批处理和流式计算了。下篇文章会单拎出来写的。
***
## 伪分布式的安装

#### 因为我们是实验项目,所以公司没有服务器给我们，我用的还是我自己的电脑,目前我的想法是,先把整个生态环境的东西都串起来然后再以集群的方式展示。所以现在都是单机版的之后会迭代集群的。

### 首先需要准备的就是二进制或者是源代码的包:[传送门](https://hadoop.apache.org/releases.html),然后就开始搭建了，不要眨眼睛。一共是四个重要的文件.位于:$HADOOP_HOME/etc/hadoop/ 下,需要修改的是：

#### core-site.xml：(注意已经以下所有代码已经携带根标签，无需再添加)

```
<configuration>
<!-- 指定HDFS老大（namenode）的通信地址 -->
   <property>
       <name>fs.defaultFS</name>
       <value>hdfs://localhost:9000</value> # 这个地方的ip,不知道填什么的请看下方【补充】内容。
   </property>
<!-- 指定hadoop运行时产生文件的存储路径 -->
   <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/buildData/hadoopFile</value>
   </property>
</configuration>
```

#### hdfs-site.xml:

```
<configuration>
<!-- 设置hdfs副本数量 -->
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
```

#### yarn-site.xml:

```
<configuration>

<!-- Site specific YARN configuration properties -->
<!-- reducer取数据的方式是mapreduce_shuffle -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>localhost</value>
  </property>

</configuration>
```

#### mapred-site.xml:

```
<configuration>
<!-- 通知框架MR使用YARN -->
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

#### 然后需要在$HADOOP_HOME/sbin/ 下修改start-dfs.sh、stop-dfs.sh、

```
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root
```

#### 以及start-yarn.sh、stop-yarn.sh

```
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
```

#### 然后需要将Hadoop的环境变量配置好(要将/bin、/sbin都纳入环境变量之中,不过要看你用哪个。然后source /etc/profile生效)

#### 生成rsa公钥密钥,以下命令依次执行即可

```
cd ~/.ssh
rm known_hosts
ssh-keygen -t rsa
ll (此处看到下方有两个文件即可:id_rsa、id_rsa.pub)
```

#### the last one step(无报错信息代表启动成功)

```
#启动hdfs
start-dfs.sh (因为已经配置了环境变量，此命令无论在哪里执行都生效)
start-yarn.sh (同上)
```

#### 启动之后用**jps命令**查看成功之后所启动的服务(如图代表启动成功)

![输入图片说明](https://user-gold-cdn.xitu.io/2019/3/28/169c2528f757e3c2?w=368&h=170&f=png&s=20355 "WX20190227-163249@2x.png")

## 补充

```
hosts文件中的格式
第一部份：网络IP地址。

第二部份：主机名.域名，注意主机名和域名之间有个半角的点。

第二部份：主机名(主机名别名） ，其实就是主机名。

当然每行也可以是两部份，就是主机IP地址和主机名；比如 192.168.20.88 bigData
```

# 问题总结

 **No.1:再第一次启动之前，执行过格式化命令，然后启动了之后又执行了一次格式化命令**

 _问题描述:一般由于多次格式化NameNode导致。在配置文件中保存的是第一次格式化时保存的namenode的ID，因此就会造成datanode与namenode之间的id不一致。_ 

![二话不说看图](https://user-gold-cdn.xitu.io/2019/3/28/169c2528f76cd408?w=1318&h=1158&f=png&s=361103 "WX20190227-163110@2x.png")
 
#### 进入这个文件然后复制当中的cluster_id

![输入图片说明](https://user-gold-cdn.xitu.io/2019/3/28/169c2528f78bde04?w=2138&h=890&f=png&s=280861 "WX20190227-180403@2x.png")

#### 将这个id复制到你core-site.xml中修改的hadoop.tmp.dir标签的路径/dfs/data/current 下的VERSION文件(如图)

![输入图片说明](https://user-gold-cdn.xitu.io/2019/3/28/169c2528f774b548?w=774&h=224&f=png&s=37667 "WX20190227-180924@2x.png")

#### 然后重新启动服务即可
***
## 集群版
敬请期待~
