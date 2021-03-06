---
layout: default
published: true
---

# Hadoop统计平台——MMStat（五）  

## 统计类型生成规则  

上一篇文章里面提到一个组件——StatTypeTraits`<`PhpApiLogRecordPtr`>`::GetStatTypes(logRecord)。在玩模板编程的时候我认为所有函数都可以称为一个组件，当然其实这只是个人对系统的理解，并非一定要如此接受。这个组件很重要，他决定一条日志记录到底要参与那些统计项的统计，而本篇就是要说说这东西的设计思路。  

在刚开始的时候每个统计项几乎都是又一个请求的method来决定。所谓method就是url上的拼接而已，例如twitter/popular、twitter/latest等等。事实上大部分的统计类型都可以通过method来唯一判断，但MMStat面对现实环境中更复杂的问题。追溯到以前，第一个例子就是单个宝贝浏览页的PV和设备统计，对于单个宝贝的浏览在过去一直没有发送专门用于统计的请求，但就有一个twitter/reply_feed的请求会发上来，这个请求的用处是获取该宝贝的评论列表，而且属于ajax请求，一次不会请求所有数据，因此当url参数offset=0的时候，就表示第一次请求。基于这样的背景，也就是单个宝贝浏览页面中可能会请求多次twitter/reply_feed（伴随不同的offset参数），而统计程序应该总根据offset=0来判断。  

在python统计系统里面最初就是没有考虑到这样的问题，但这也很正常，因为没有必要为认为没可能出现的情况引入设计复杂性，但是当这个情况第一次被发现之后，就相当于是敏捷开发里面的“中了第一发子弹，目标就是不要再中第二发”。在这个时候我需要一个判断的逻辑，而以后可能会有更多各种各样的判断规则，因此不适合直接在代码里面嵌入if-else。  

另外还有一个问题就是当时重构要考虑的，到底什么信息会影响一条日志记录属于哪些统计类型，显然我不能因为当前问题是由url参数引发的，就把设计只扩展到url参数，从概念上来讲，日志记录里面所有信息都应该能影响判断，因此获取统计类型的接口应该足够“寛”，让整个日志记录作为参数。但这样其实还会有一点问题，那就是如果决定因素超出单个日志记录的信息，而是与会话中所有日志记录有关那怎么办？首先我在当前MMStat的设计里没有涵盖这一点，因为我认为这种可能性太少，涵盖他太复杂；其次，如果真的产生了这情况，也可以通过一个简单的预处理来解决，例如先遍历会话中所有日志记录，把条件信息重写到某条日志记录中，那么依然可以通过单条日志记录来判断统计类型，至于这个遍历逻辑放到哪里，这可以再考虑，反正可以固化GetStatTypes的代码了。  

下面看一下GetStatTypes的设计图：  
![统计项规则表](/assets/stat_type_rule_table.png)  

统计项规则表是基于一个已有的情况设计出来的——大部分的统计项都可以通过method来唯一判断，而所谓method其实就是字符串而已，也就是说总能通过字符串来找到匹配的统计项集合。在表里面看到每一个key背后都可能有多个统计项，具体点的例子就是：单个宝贝浏览请求既属于“单宝统计”，也属于“全部统计”；例如新浪互联注册请求属于“注册统计”，也属于“新浪微博注册统计”，还属于“互联注册统计”。而对每一统计项还配有一个判断器，那是用来判断该日志记录到底是不是属于该统计项，因此并非说key值满足了，就可以直接关联多个统计项，每个统计项还有自己的判断器，这非常有用，尤其是在不同版本的app和api之间，同样的请求可能有不同的统计项集合来对应。虽然这看起来很灵活，但其实一个架构好的系统不应该出现这么混乱的情况，但我没办法控制那混乱的api和app架构，所以只能让MMStat更灵活。  

然而GetStatTypes组件不只是对统计项规则表的包装，更重要是对于那些比较复杂的判断逻辑，可能用在规则表中不自然的逻辑，GetStatTypes还提供了调用这些复杂判断逻辑的空间，因此外部只看到一个函数，但内部能进行复杂的判断，并返回最终该日志要参与哪些统计。