---
layout: default
published: true
---

# Hadoop统计平台——MMStat（三）  

## 数据存储视图  
  
这一篇讲的是MMStat的数据文件存储视图，以及目前的算法是如何输入和输出数据到HDFS上。理解这个视图对理解整个系统很有帮助，从第一篇可以看到，MMStat的运作模式其实都是从HDFS拿数据，然后map-reduce处理，然后又把数据存储到HDFS上。  

