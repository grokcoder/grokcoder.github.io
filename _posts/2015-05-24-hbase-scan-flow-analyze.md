---
layout: post
title: "HBase scan flow analyze"
date: 2015-05-24 13:51:20
author: Oliver
categories: ['HBase']
tags: ['源码分析']
---


###前言
本文主要就HBase的scan操作在客户端阶段的原理进行相关的分析，HBase的查询主要是用过scan的操作进行的，scan的操作室指对HBase数据库的满足响应的条件的相关数据进行扫描操作，并返回KeyValue形式的Result.典型的scan的操作流程如下面的代码所示：
{% highlight java %}
TableName tn = TableName.valueOf("test1");
Configuration conf = HBaseConfiguration.create();
try (Connection connection = ConnectionFactory.createConnection(conf)) {
   			try (Table table = connection.getTable(tn)) {
                    Scan scan = new Scan("1".getBytes());
                    //scan.setAttribute("type", "parquet_scan".getBytes());
                    ResultScanner rs = table.getScanner(scan);
                    Iterator<Result> it = rs.iterator();
                    while (it.hasNext()){
                        Result result = it.next();
                        while (result.advance()){
                            Cell cell = result.current();
								...
                        }
                    }
                }
     }
{% endhighlight java %}
上述的例子比较简单，scan只是设置了一个startRow的值，这样的查询能够将数据库中以startRow开始的后续部分的数据都能够查询出来，当然scan能够对其查询数据进行进一步的范围约束，主要由以下几种常见方式：

<ol>
	<li>设置startRow和endRow查询两者之间的数据</li>
	<li>addFamily 查询对应的列族</li>
	<li>addColumn 查询具体的列</li>
	<li>addFilter 通过各种过滤器进一步进行删选</li>
	<li>设置scan.issmall = true(这个是对小数据的查询，比如meta信息)</li>
</ol>

接下来就继续对scan的操作流程进行分析。

##Scan流程分析之初始化

从代码1中可以看出scan操作核心部分就是通过一个ResultScanner的类进行不断地迭代拿到每个Result类型的数据，首先分析`table.getScanner(scan)`所触发的相关操作。该操作实际上是调用了HTable得getScanner得操作，Table类只是一个借口，该函数返回的实际对象是ClientScanner,如下代码所示：

{% highlight java %}
return new ClientScanner(getConfiguration(), scan, getName(), this.connection,
          this.rpcCallerFactory, this.rpcControllerFactory,
          pool, tableConfiguration.getReplicaCallTimeoutMicroSecondScan());
{% endhighlight java %}
这里有必要将ClientScanner的继承体系关系介绍一下，Scanner所涉及的类图如下：


 <img src="/static/post_img/hbase-client-scan.png" alt="替代文本" title="scan-flow" width="720" height = "370" />
 

从改图可以看出，scan的操作所涉及的类主要有两种，一个是Scannner一个是Callable,其中Scanner实际上负责对取出来的数据的遍历，而callable则实际负责从服务器端取数据。

1.new ClientScanner发生的事情
首先，同普通的类初始化相同设置一些基本的参数，这里值得注意的时caching的值，这个是设置scan的cache大小，如果默认值为100，也可以在hbase-site.xml设置hbase.client.scanner.caching的值。
其次是结束基本变量赋值之后的initializeScannerInConstruction，该方法实际调用了` nextScanner(int nbRows, final boolean done)`,进行scanner的初始化

这里有几个步骤：  
(1) 关闭之前的scanner如果他们是open的话  
(2) 下一个scanner的startkey的确定，如果currentRegion ！= null的情况下，startkey 为当前region的endkey,否则为scan.startRow  
(3) 初始化callable（ScannerCallableWithReplicas）  
(4) 打开scanner(默认nbRows为this.caching = 100)  
代码如下：

{% highlight java %}
  protected boolean nextScanner(int nbRows, final boolean done)
    throws IOException {
      // Close the previous scanner if it's open
      if (this.callable != null) {
        this.callable.setClose();
      }

      // Where to start the next scanner
      byte [] localStartKey;

      // if we're at end of table, close and return false to stop iterating
      if (this.currentRegion != null) {
        byte [] endKey = this.currentRegion.getEndKey();
        ...
        localStartKey = endKey;
      } else {
        localStartKey = this.scan.getStartRow();
      }
        callable = getScannerCallable(localStartKey, nbRows);
        // Open a scanner on the region server starting at the
        // beginning of the region
        call(scan, callable, caller, scannerTimeout);
        this.currentRegion = callable.getHRegionInfo();
    }
{% endhighlight java %}

接下来我们继续看看call语句发生了什么，继续跟进代码，我们会发现其最终是调用代码：

{% highlight java %}
	if (scannerId == -1L) {
        this.scannerId = openScanner();
      }
{% endhighlight java %}
  这其实是向服务器端发送一个打开scanner的请求，继续跟进服务端代码：
服务端的处理函数是RsRpcServices.java中的``` ScanResponse scan(final RpcController controller, final ScanRequest request)```
服务器会根据request中的scannerid去决定是打开一个新的scanner还是直接获取缓存的scanner。

初始scanner会按照以下的树状图中的Scanner层级依次初始化.RegionScanner对应的HReion对相应scan地查询的一个管理Scanner,而StoreScanner对应一个HStore,MemStore对应对MemStore的查询，而StoreFileScan则对应HFile文件的查询。
初始化结束后，会将打开的scanner缓存，并返回一个scanner id给客户端，客户端随后就能够通过该id进行scan的数据查询服务

##Scan 流程分析之数据流分析

如下图所示，是数据请求的基本流程，左上角是我们scan的客户端的主要代码，在客户端代码中每次的hasNext();  
1.请求首先是查看本地的cache中是否还有数据，如果还有的话那么直接在cache中取出数据并返回;  
2.否则就会委托ScannerCallable进行一次RPC的请求，这时候，服务器端就根据scanner id来获取对应的服务器端对象进行数据的读取并返回（读取数据量的大小由scan参数控制），然后将数据放到本地缓存  
3.重复1-2 直到所需数据全部取出。  

 <img src="/static/post_img/scan-flow.png" alt="替代文本" title="scan-flow" width="720" height = "370" />
 
 至此，HBase Scan的流程分析就结束了，其实总结一下就几个关键步骤：  
 (1)按照row的分布定位到对应的HRegion  
 (2)为对应HRegion打开RegionScanner  
 (3)数据查询  