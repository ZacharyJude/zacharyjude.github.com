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
每个processor组件都要对组织session的reducer实现两个接口——Process和Finish，其中Process接口要接受被组织好的session数据结构，并且不能修改session任何数据；而Finish就是在reducer接收完所有分组后，并且即将要结束reduce过程的时候调用。只要工程师实现Processor的时候实现了上述的两个接口，而且不修改reducer的逻辑，那么代码就可以保证正确执行。  

在MMStat的设计里面我几乎没有用到继承（就写这篇文章的时候还没用过），都是基于模板的抽象和组件设计。我曾经是非常崇尚面向对象设计的人，在大学期间我非常沉迷各种设计模式和其应用，并非说OOP和设计模式有什么比不上模板编程的重大缺点，只是在一个复杂系统里面，我更趋向于在编译过程中发现设计上的误解，而不是到了运行时。毕竟像我所在的那种团队不可能开发出像.NET、JAVA这样设计得很好的平台组件库（我相信也没几家公司认为他们可以），基于这一点，根据我自己的经验，OOP中最核心的继承在实际工程环境中给工程师埋的坑远大于修出来的好路。  
当工程师写一个类，满怀兴奋地去继承前人写的类的时候，中间会有多少前人意料之外的设计误解，这都必须要等到系统实际运行起来才知道，更危险的是刚开始的时候运行起来也没问题，于是越来越多对该类的继承依赖，直到某天开发某个重要功能的时候才发现原来一直对最初继承的那个类有设计上的误解，那就成了最头痛的事情。这一篇文章不是专门用于讨论我对模板编程和面向对象编程的关系的，因此就不在这里多说了。或者以后能写这么一篇文章出来嗨皮一下，所以下面就回到正题。  
  
ClassifySessionDataProcessor是生成分类元数据的组件，也是最基础的组件，下面是他内部的组件图：  
![组织session reducer的组件图2](/assets/component_session_reducer2.png)   
  
ClassifySessionDataProcessor包含两个map（我称他做统计表），这个map的key是统计项名称，一个字符串，而对应的value就是一个counter组件，这个组件就负责针对这个统计项的统计。当这个processor处理session中每一条日志记录的时候，会获取每条日志记录参与的统计项名称集合，然后从统计表中检索每个统计项是不是要统计，如果要统计的话就取出对应的counter并进行统计调用，如下图所示：  
![统计项与日志记录的关系](/assets/stat_entry_log_record_relation.png)    
  
为什么不是全部统计项都用一个统计表来关联呢？目前发现有三个可能的原因：  
1.  逻辑上功能和目标不一样，可能一个表是为了统计一类功能，而另一个表则为是另一统计功能。强行放在一齐是不好的设计。尤其是这种结构还可以用到其他processor上。  
2.  针对不同的统计项可能使用类型不一样的counter，因为统计表就是一个STL里面的map，当采用不一样的counter时，自然就不能兼容到同一个map类型里面。当然，如果不同的counter继承一个公用的接口就能解决这个问题，但OOP的问题之一就体现在这里，是不是为了解决这么一个问题就引用需要维护的继承，难道counter的设计初衷就是为了被统计表使用吗？显然不是，甚至说counter之间其实没什么必要有关联，但为了解决这个问题可能就得让他们都继承于同一个接口；当然这时候坚持OOP的话还是可以用适配器模式来来保证counter的设计没有问题，只是适配器类都继承相同的接口，这或许是个好的设计，但我还是觉得没必要用那么复杂的设计。当然使用模板也不意味着counter设计之间可以完全没有关系，但这种关系就是他们要有同样的函数（在具体里面是一个签名完全一样的FeedElement函数），但如果没有实现这个函数的话，在编译的时候就会报上错误。  
3.  不同的统计表可以关联相同的统计项，虽然在目前我还没碰到这种情况，但保持设计的灵活也是必须的。可能读者也会觉得就算用一个表也可以做到这一点，因为一个统计项名称可以对应一个counter列表，但这样会违背一个原则，就是保持设计简单，这样的话统计表的结构会变得要复杂一点，但其实这种复杂完全可以分担在外部，在做很多组件设计的时候我都会考虑到这一点，让组件设计保持简单，设计更多组件各自分担一点复杂度。
  
接下来要说明的就是两个基础的组件，也是第一幅示意图里面的ClassifyCounter和BasicCounter。如果是写上模板参数也不会很长的组件，我就直接用代码形式来写模版信息了，但如果写出来比较长的话，我就用示意图来表示，当然这跟我个人喜欢作图有关系，呵呵。  
BasicCounter`<`TElement`>`是一个用于计数的组件，他目前支持这几种计数：  
1.  总计数  
2.  去重计数  
3.  出现次数计数
  
BasicCounter的主要接口就是一个FeedElement函数，下面是他的实现，这个接口就是用于统计给定元素：  

    BasicCounter的FeedElement代码  
	void FeedElement(const TElement& elem) {  
	    this->_feedElementTimes++;  
	    if(this->_isEnableUniqCount) {  
		this->_uniqOccur.insert(elem);  
	    }  
	    if(this->_isEnableOrderedCount) {  
		this->_orderedOccur.push_back(elem);  
	    }  
	    if(this->_isEnableOccurCount) {  
		typename map< TElement, TStatInt >::iterator findIter;  
		findIter = this->_countOccur.find(elem);  
		if(this->_countOccur.end() == findIter) {  
		    this->_countOccur[elem] = 1;  
		}  
		else {  
		    (findIter->second)++;  
		}  
	    }  
	    return;  
	}  

通过模板，BasicCounter可以用于计数不同的类型，因为内部使用STL来维护这些要计数的类型，因此模板类型也必须要能够用于STL中。在构造BasicCounter的时候需要指定要计算那些类型的计数，默认是全部都会计算，但一般是不需要的。  
ClassifyCounter`<`TElement, TBaseCounter`>`是用于分类统计的基础组件。为什么需要这个组件？这个可以回溯到第二篇文章里面提到的维度分类，因为有了维度分类统计的概念，自然就需要在代码层面有一个可以高度复用的概念。两个模板参数分别是输入的元素类型，和子counter的类型。对ClassifyCounter的简单描述就是：**对输入元素进行分类，并用所属分类下的TBaseCounter对TElement的某些数据进行统计**。不过这样的描述还是太抽象了，所以还是上示意图：  
![ClassifyCounter工作原理](/assets/classify_counter_work.png)  
  
如图所示，每个ClassifyCounter在构造的时候需要配置三个组件：分类器、属性获取器和子Counter构造器。这三个组件都是由外部定制的，因此ClassifyCounter的本质就是固化和高度抽象了一个分类统计的行为。注意这里我没有让TBaseCounter直接就是BasicCounter，因为这不是唯一绑定的关系，TBaseCounter甚至也可以是ClassifyCounter。下面是ClassifyCounter的关键代码：  

    ClassifyCounter的FeedElement代码  
	void FeedElement(const TElement& classifyTarget) {  
	    this->_bufferForHoldClassify.clear();  
	    bool canClassify = this->_classifier(classifyTarget, this->_bufferForHoldClassify);  
	    if(!canClassify) {  
		return;  
	    }  
	    BaseElementType element = this->_elementGetter(classifyTarget);  
	    for(TSetS::const_iterator iterClassify=this->_bufferForHoldClassify.begin();  
		iterClassify!=this->_bufferForHoldClassify.end();  
		++iterClassify) {  
		TClassifyMapperIter findIter = this->_mapper.find(*iterClassify);  
		if(this->_mapper.end() == findIter) {  
		    BaseCounterPtr newCounter = this->_baseCounterCreator();  
		    newCounter->FeedElement(element);  
		    this->_mapper[*iterClassify] = newCounter;  
		}  
		else {  
		    findIter->second->FeedElement(element);  
		}  
	    }  
	}  
  
从代码可以看出_baseCounterCreator总是一个没有函数的对象生成器，但不可能在现实环境里面总是写出没有构造参数的组件，是吧？函数式编程的力量就在于参数的绑定，因此在实际应用的时候，这里的_baseCounterCreator几乎都是已经经过参数绑定的函数对象。  
