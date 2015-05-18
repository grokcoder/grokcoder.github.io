---
layout: post
title: "CreateTable 的服务器端实现分析 "
date: 2015-05-18 15:20:13
author: Oliver
categories: ['HBase']
tags: [源码分析]

---

HBase的所有请求调用都是通过RPC的机制进行的，RPCServer监听到请求之后会解析请求内容，然后根据解析的方法以及参数调用服务器端实际的方法，这也是远程代理模式的经典做法，createTable的请求最终实现是在HMaster中的，但是实际的表的建立过程是在CreateTableHandler类中的，接下来主要就HBase中表的建立过程进行详细分析。

## 1. HMaster的createTable实现

如下代码所示，是HMaster中的createTable的流程代码：

{% highlight java %}
    public void createTable(HTableDescriptor hTableDescriptor,
                            byte[][] splitKeys) throws IOException {
        if (isStopped()) {
            throw new MasterNotRunningException();
        }

        String namespace = hTableDescriptor.getTableName().getNamespaceAsString();
        ensureNamespaceExists(namespace);

        HRegionInfo[] newRegions = getHRegionInfos(hTableDescriptor, splitKeys);
        checkInitialized();
        sanityCheckTableDescriptor(hTableDescriptor);
        if (cpHost != null) {
            cpHost.preCreateTable(hTableDescriptor, newRegions);
        }
        LOG.info(getClientIdAuditPrefix() + " create " + hTableDescriptor);
        this.service.submit(new CreateTableHandler(this,
                this.fileSystemManager, hTableDescriptor, conf,
                newRegions, this).prepare());
        if (cpHost != null) {
            cpHost.postCreateTable(hTableDescriptor, newRegions);
        }

    }
{% endhighlight %}

在正式创建表之前做的几件事情：
1.检查Master是否正常运行
2.检查索要创建的表的`namespace`是否存在
3.`HRegionInfo`类包含了HRegion的相关信息getHRegionInfos(),函数按照splitKeys和表描述信息，获取该表对应的HRegion的信息。

>这里有必要解释一下`HRegion`，`HRegion`存储了table的数据信息，它包含了每一个row的所有columns,一个table包含1到多个hregion,每一个hregion包含多个`HStores`,每个HStores包含一部分的Rows和对应部分的Columns。
每个HRegion包含了一个[startKey, endKey),用来标识其保存的row的范围，英雌HRegion可以由`TableName`和`key range`唯一确定

4.检查Master是否完成初始化
5.检查表信息是否符合规定
6.建表，建表又分为三个过程：

	1. cpHost.preCreateTable(hTableDescriptor, newRegions);
	2. submit(new CreateTableHandler(this,
                this.fileSystemManager, hTableDescriptor, conf,
                newRegions, this)
    3.cpHost.postCreateTable(hTableDescriptor, newRegions);
  
  
其中步骤1和3都是为了协处理器预留的钩子函数，方便应用开发人员动态添加新的功能。
接下来主要分析一下步骤2中的建表所做的操作

## 2. CreateTableHandler 中的建表实现

其实在代码`this.service.submit(new CreateTableHandler(this,
                this.fileSystemManager, hTableDescriptor, conf,
                newRegions, this).prepare());`中调用的过程分为两个部分，一个是prepare,然后才是submit，先来看一下prepare（）的工作

###2.1 建表之前的准备工作prepare()分析

{% highlight java %}
  public CreateTableHandler prepare()
      throws NotAllMetaRegionsOnlineException, TableExistsException, IOException {
    // Need hbase:meta availability to create a table
    try {
      if (server.getMetaTableLocator().waitMetaRegionLocation(
          server.getZooKeeper(), timeout) == null) {
        throw new NotAllMetaRegionsOnlineException();
      }
      // If we are creating the table in service to an RPC request, record the
      // active user for later, so proper permissions will be applied to the
      // new table by the AccessController if it is active
      if (RequestContext.isInRequestContext()) {
        this.activeUser = RequestContext.getRequestUser();
      } else {
        this.activeUser = UserProvider.instantiate(conf).getCurrent();
      }
    }
    //acquire the table write lock, blocking. Make sure that it is released.
    this.tableLock.acquire();
    boolean success = false;
    try {
      TableName tableName = this.hTableDescriptor.getTableName();
      if (MetaTableAccessor.tableExists(this.server.getConnection(), tableName)) {
        throw new TableExistsException(tableName);
      }

      checkAndSetEnablingTable(assignmentManager, tableName);
      success = true;
    } finally {
      if (!success) {
        releaseTableLock();
      }
    }
    return this;
  }
{% endhighlight %}
这里的主要工作如下：
1. 获取hbase:meta 信息，meta是hbase的一个特殊表，其存数了HBase上面的RegionServer的信息以及其分布情况。
2. 检查创建表的用户的权限并记录。
3. 获取table write锁
4. 检查表是否存在，已存在则抛出异常
5. 为了防止多个线程发起建同一个表的情况，可以在建表未成功之前可以先设置该table enable,这样其他线程就不能再建表。

建表之前的准备工作到此结束，一下分析具体建表流程

### 2.2 建表具体实现handleCreateTable过程分析

handleCreateTable,其实主要做了三件事，1.在磁盘上建表，2.meta表，3.为新建的表分配对应的regionserver。详细代码如下：
这块的代码是分为八个小步骤，我们一一分析，

1.创建表描述符

{% highlight java %}
// 1. Create Table Descriptor
Path tempTableDir = FSUtils.getTableDir(tempdir, tableName);
new FSTableDescriptors(this.conf).createTableDescriptorForTableDirectory(
tempTableDir, this.hTableDescriptor, false);
Path tableDir = FSUtils.getTableDir(fileSystemManager.getRootDir(), tableName);

{% endhighlight %}

首先创建一个临时的文件夹，然后创建对应的文件表描述符，最后创建该表在文件系统中的路径，
>HBase的表的数据对应在文件系统中的一个文件夹下，该文件夹也就是表名

2.创建Regions

{% highlight java %}
// 2. Create Regions
List<HRegionInfo> regionInfos = handleCreateHdfsRegions(tempdir, tableName);
{% endhighlight %}

为table 创建in-disk数据结构，内部具体创建了存储table数据的HRegion，并返回hregioninfo的信息

3.将步骤1中的临时文件夹移到HBase的根目录下，如果hregions创建成功的话，继续一下几个步骤：

{% highlight java %}
if (regionInfos != null && regionInfos.size() > 0) {
// 4. Add regions to META
	addRegionsToMeta(regionInfos);
// 5. Add replicas if needed
	regionInfos = addReplicas(hTableDescriptor, regionInfos);
// 6. Trigger immediate assignment of the regions in round-robin fashion
	ModifyRegionUtils.assignRegions(assignmentManager, regionInfos);
}
{% endhighlight %}

4. 将新建的hregion信息注册到hbase的meta表中；
5. 有必要的话创建这些hregion的副本
6. 为新建的hregions分配对应的regionserver
7. 在zookeeper中将新建的table设置为enable状态：

{% highlight java %}
// 7. Set table enabled flag up in zk.
try {
    	assignmentManager.getTableStateManager().setTableState(tableName,
        ZooKeeperProtos.Table.State.ENABLED);
    } catch (CoordinatedStateException e) {
      throw new IOException("Unable to ensure that " + tableName + " will be" +
        " enabled because of a ZooKeeper issue", e);
    }
{% endhighlight %}
8.跟新tabledescripter cache

{% highlight java %}
// 8. Update the tabledescriptor cache.
((HMaster) this.server).getTableDescriptors().get(tableName);
{% endhighlight %}

至此服务器端数据库表的建立过程源码分析结束