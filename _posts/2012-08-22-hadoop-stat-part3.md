---
layout: default
published: true
---

# Hadoop统计平台——MMStat（三）  

## 组织session并生成基础元数据
  
这一篇讲的是MMStat统计计算的第一个流程——组织session并生成基础元数据，正如上一篇所说，元数据是统计的基础，而元数据的产生又依赖于用户行为session的组织，因此两个过程可以认为是捆绑在一齐，但其实这中间还能涉及更多，在下面会提到。  

下图是组织session过程的map-reduce流程的文件级数据流图：  
![组织session的文件数据流图](/assets/organize_session_file_data_flow.png)  
  
组织用户行为session是统计计算的第一步，这一步要达到两个目的——1.把属于同一个用户的日志聚合起来，2.把这些记录按***服务器**访问时间排序。
