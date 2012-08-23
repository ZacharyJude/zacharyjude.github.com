---
layout: default
published: true
---

# Hadoop统计平台——MMStat（三）  

## 组织session并生成基础元数据
  
这一篇讲的是MMStat统计计算的第一个流程——组织session并生成基础元数据，正如上一篇所说，元数据是统计的基础，而元数据的产生又依赖于用户行为session的组织，因此两个过程可以认为是捆绑在一齐，但其实这中间还能涉及更多，在下面会提到。  

下图是组织session过程的map-reduce流程的文件级数据流图：  
![组织session的文件数据流图](/assets/organize_session_file_data_flow.png)  
  
组织用户行为session是统计计算的第一步，这一步要达到两个目的——1.把属于同一个用户的日志聚合起来，2.把这些记录按**服务器**访问时间排序。根据上一篇提到，如果只是分类元数据的话，其实没有必要组织session，因为每一条日志记录都可以**独立地**解析出其维度信息（app、device和version等）和统计内容（device token、imei、user id等），因此在python统计系统里面，生成分类元数据这个过程在刚设计的时候是直接进行的，跳过了组织session这一步。  
但在现实环境中这样理想的“独立性”是不存在的，例如美丽说app曾经在三个版本中由于客户端的BUG导致同一个海报页一次请求会连续发多遍，在这样的情况下当时leader给出了一个需求就是让同一个设备在同一秒内连续发出的三个同样的请求按一个来算。很明显在这样的需求下，一条日志记录是否要参与生成元数据，就不再独立了，而是依赖于上下文来判断。这样的情况以后必然会继续出现，因此先组织session再做后续的事情是理所当然的。  
下面是组织session的mapper非常简单的工作流程：  
![组织session的mapper工作流](/assets/organize_session_mapper_workflow.png)    


