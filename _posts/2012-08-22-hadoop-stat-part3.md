---
layout: default
published: true
---

# Hadoop统计平台——MMStat（三）  

## 组织session
  
这一篇讲的是MMStat统计计算的第一个流程——组织session，正如上一篇所说，元数据是统计的基础，而元数据的产生又依赖于用户行为session的组织，因此两个过程可以认为是捆绑在一齐，但其实这中间还能涉及更多，在下面会提到。  

下图是组织session过程的map-reduce流程的文件级数据流图：  
![组织session的文件数据流图](/assets/organize_session_file_data_flow.png)  
  
组织用户行为session是统计计算的第一步，这一步要达到两个目的——1.把属于同一个用户的日志聚合起来，2.把这些记录按**服务器**访问时间排序。根据上一篇提到，如果只是分类元数据的话，其实没有必要组织session，因为每一条日志记录都可以**独立地**解析出其维度信息（app、device和version等）和统计内容（device token、imei、user id等），因此在python统计系统里面，生成分类元数据这个过程在刚设计的时候是直接进行的，跳过了组织session这一步。  
但在现实环境中这样理想的“独立性”是不存在的，例如美丽说app曾经在三个版本中由于客户端的BUG导致同一个海报页一次请求会连续发多遍，在这样的情况下当时leader给出了一个需求就是让同一个设备在同一秒内连续发出的三个同样的请求按一个来算。很明显在这样的需求下，一条日志记录是否要参与生成元数据，就不再独立了，而是依赖于上下文来判断。这样的情况以后必然会继续出现，因此先组织session再做后续的事情是理所当然的。  
下面是组织session的mapper非常简单的工作流程：  
![组织session的mapper工作流](/assets/organize_session_mapper_workflow.png)    

如图所示，mapper做的事情就是先把日志解析一次，提取每条日志中的设备ID，然后以设备ID为KEY来输出数据。map的过程很简单，接下来是reduce过程的示意图：  
![组织session的reducer工作流1](/assets/organize_session_reducer_workflow1.png)  
上图主要是展示Hadoop shuffle的效果，而且值得一说的是，并非所有经过shuffle之后的reduce输入数据文件都只有一个设备ID作为KEY，只是图上只展示一个设备ID而已，只要两个设备ID经过partition算法后（默认使用全键HASH）得出相同的值，就会落到同一个reduce输入数据文件中，并且按设备ID排序好。下面是reduce过程的简述：  
![组织session的reducer工作流2](/assets/organize_session_reducer_workflow2.png)  
  
图中有几个点值得一提：  
1.  reducer采用分组的方式来进行数据处理，在设计reduce过程的时候千万不能想着先把所有输入数据都缓存下来，然后进行处理，因为根本无法预知会有多大的输入数据。在刚开始的时候我就是很单纯地把所有设备ID对应的数据都接受了，然后再处理，而当时完全没有问题，因为一天的数据量根本不大，一旦变成多天输入，那就很可能会压垮内存；合理的做法是分组来进行处理，因为Hadoop已经给输入数据按照key来排序好，如果当前输入的key跟上一次输入的key不一样，那就意味着一个分组的输入已经结束了。那会不会存在分组处理跟集中处理结果不一样的情况呢？按我的理解这是非常罕见的情况，因为本来从map到reduce经过的shuffle过程就无法保证不同的key会落到同一个reducer的输入数据中（这完全又partition算法来决定，虽然你可以配置一个自定义的partition算法）。正常情况下属于同一个reducer输入数据的各组key之间应该是没有什么关系的，哪怕有，也只是优化性能上的关系。  
2.  对日期进行了划分存储，因为在统计时几乎都是按天来结算，因此就算是同一个设备，不同日期的日志集合不能算是一个session，必须要是同一天内的。  
3.  扩展点。注意到在组织好session之后，会把每一个session（其实就是排序好的日志记录集合）交给processor进行处理，所以processor就是这里的扩展点，目前最重要的就是生成分类元数据的processor了。  
  
下一篇文章会讲到这个过程中的详细设计，也就是里面每一个组件之间是如何协作的。
