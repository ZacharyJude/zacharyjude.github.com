---
layout: default
published: true
---

# Hadoop统计平台——MMStat（四）  

## 组织session和生成分类元数据--详细设计  
  
上一篇从数据流的角度讲述了组织session的map-reduce过程，而这一篇讲的就是这个过程的详细设计——组件之间如何配置、使用和协作从而稳定工作起来。  

从上一篇可以看到，组织session的mappe流程很简单，所以重点在于reduce过程，下面是reduce过程中参与流程的主要组件协作图：  
![组织session reducer的组件图1](/assets/component_session_reducer1.png)  
  
如图，reducer层次的组件结构很简单，就是一个reducer拥有两个map，每个map都是用日期字符串来做索引，根据日期字符串会各自有一个processor。组织session的reducer设计是可以处理多天的输入，而每个设备每天的session都应该有一个独立的processor来处理。这里可能会有一个疑问是，为何要每个日期对应一个processor，不能就用一个吗？的确是可以的，这就相当于在processor内部来维护分日期的计算，但这样可能会加大processor的设计复杂度，因此我选择在外部来做这个索引，简化processor，毕竟有一点很重要，那就是processor是功能扩展点，越是简化对processor的要求，功能开发会越快。  
当reducer每次接收完一个分组（一个设备号）的数据，就会组织session，最后把组织好的session交给属于当天的processor来处理。  


