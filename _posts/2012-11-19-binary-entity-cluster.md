---
layout: default
published: true
---

# 二元组实体（数据）聚类  

以前没有接触过聚类的工作，这是我第一次做聚类，在显示环境下的数据聚类要得出理想的效果，其难度远远大于我原来的想象，在过去一周我无数次因为不知所措而感到很沮丧，直到现在才能做出相对有效的聚类，觉得有必要为这一周多的苦逼写下一篇记录。  

## 背景
跟标题一样，这个聚类过程的输入是一对一对的二元组数据，在我的工作里面是(userID, shopID)，表示用户喜欢过这个商店。而聚类的期望目标就是按照把商店按照用户的喜好特点来聚类，也就是每个聚类中的商店，都具有代表一类用户偏好的特征。刚开始给出的目标是对(用户，商店)这个二分图进行聚类，同时获得用户的聚类{Ui}和商店的聚类{Si}，使得Ui里的用户都喜欢Si里面的商店。在会议室讨论这个聚类的时候，Austin认为这种二分图聚类的问题在业界应该是被研究得很透彻，因此应该有现成的解决方案，而会后，他的确给我找到了这样的一篇论文，而我在这个事情上的工作就从这一片论文正式开始。之所以说正式开始，因为在会议之前其实我已经有了一定的设计，写了一个算法来做这个事情，当然用的不是论文里面的高深方法，而是下面就要说到的一个错误的方法。  

## 错误的算法
针对这个问题的输入，很容易可以构造出两个倒排索引：(shopID, userIDs)和(userID, shopIDs)。我基于一个思考：如果各自喜欢两个商店的用户集合的交集大小达到一个阀值以上，那么这两个商店就应该具有一定的共同特征，这个特征的细化程度取决于阀值的大小，阀值越大，特征越是细化。扩大商店聚类的方式就是迭代每一个已有的聚类（初始的时候就是每个商店看作一个已有聚类），如果能通过喜欢用户群交集大小的阀值，就并入到已有聚类中，形成新的聚类这个算法有以下几个问题：  
第一，这样在第一轮里面的计算量会很大，因为是个n次方的扩展（N为扩展次数），从一个店的聚类扩展到两个店，就有m\*m的可能候选聚类（m是商店的数量）；从两个店的聚类扩展到三个店，理论上就可能有m\*m\*m的处理复杂度。在一般情况下，如果阀值提高一点的话，经过第一轮扩展后，后续的扩展都不会达到很高的复杂度（通常都比第一轮扩展复杂度要低），因此性能问题不是这个算法最糟糕的地方。

第二，这个算法基于用户的交集来构造商店的聚类，而为了缓解上面第一个问题，使得程序能够在可接受的复杂度内完成计算，就必须让用户交集数量的阀值设置得比较高，在我的样本数据里面有105万用户和7800商店，在这个数据规模下经过测试我得起码把阀值设置到5000或以上才能让计算时长可接受。这样高的用户阀值会导致很严重的问题——使得由小众用户（5000以下的）构成的商店聚类在结果中消失，简单来说，因为为了让算法可执行，而丢弃了应该被注意到的聚类结果。并且在这个情况下最终计算出来的商店聚类也是基于大概5000用户的交集上，因为随着每个聚类扩展，共同的用户数量会锐减。而在实际结果下，聚类大小达到4个店铺就已经无法扩展了（也就是不存在5个店铺拥有5000以上共同的用户）。  

起初我以为在第二、三轮扩展的时候把阀值降低或许能缓解问题，没错，这样虽然可以让聚类大小增加一点（成了大小为6个聚类），但这样跟预期想要的还是差太远了，毕竟我们的聚类目标是希望，通过分析聚类中的商店共同的特征，假设成用户的爱好，然后对用户群进行划分，这样只有5，6个商店的聚类根本不能作为研究对象。这个算法通过对已有的聚类进行扩展，会得出这样的结果，{S1,S2,S3,S4}、{S1,S2,S3,S5}、{S2,S3,S4,S5}、{S1,S2,S4,S5}等等只相差一个商店的聚类结果，如果聚类大小更大的话，会产生更多这样的结果，但正如前面提到，才5、6个商店大小的聚类就无法再扩展了；基于这个情况，我选择把聚类给合并一下，顺便也通过这样扩大聚类的size，然后当我一合并就发现所有商店几乎都会merge到一起，原因是因为这些聚类之间本来相差的商店就不多，而构造过程都基于交集用户，于是最终导致一个结果——所有不同聚类中的商店其实都拥有一部分共同用户，在我的运行结果中大概是3000个。当我搞懂这一点以后，我彻底明白这个算法是错误的。  

## Leran From Papers
前面提到Austin帮我找了一份paper，叫做《Bipartite Graph Partitioning and Data Clustering》，里面给出一种算法可以同时对二分图中两边的元素进行*无重叠*聚类，并且给出了数学证明。以前我虽然看过paper，但直到这一次我真的想把paper中的知识直接应用到工程中我才发现自己的知识是多么脆弱。paper里面给出了算法的数学证明，一大堆的矩阵运算让我完全摸不着头脑，尽管我看懂了作者是根据什么来衡量聚类的相关度，但对于算法为什么正确，甚至连算法具体要计算什么，是怎么个流程我也无法清楚知道；如果不是要具体应用起来的话，我会以为自己已经大致能掌握这paper的内容，就差证明而已，但这次我就完全知道自己的天真了。  

在这份paper里面有一个核心的思想就是把这个问题的解决方式引导到解SVD的问题上，这就给我引了一个很大的谜团——什么是SVD？解决什么问题？在网上我随便都能找到很多SVD的文章啊，博客啊之类的，但我必须得承认没有一篇是能让我这种完全把线性代数还给老师的人能看懂的。最后我找到了一份23页的PDF，应该也算是一份paper，我在看过这份paper的introduction之后就认定它是我所寻找的！  

Most tutorials on complex topics are apparently written by very smart people whose goal is to use as little space as possible and who assume that their readers already know almost as much as the author does. This tutorial’s not like that. It’s more a manifestivus for the rest of us. It’s about the mechanics of singular value decomposition, especially as it relates to some techniques in natural language processing. It’s written by someone who knew zilch about singular value decomposition or any of the underlying math before he started writing it, and knows barely more than that now. Accordingly, it’s a bit long on the background part, and a bit short on the truly explanatory part, but hopefully it contains all the information necessary for someone who’s never heard of singular value decomposition before to be able to do it.

这份paper正文一共21页，但就花了13页来扫盲，把SVD涉及到的所有线性代数基础知识都集中用简单例子阐述了一遍，因此让我觉得之前看过的SVD资料都多余了，[如果一开始能找到这份paper](http://vdisk.weibo.com/s/hQiRd/1353311091)，会跳过不少痛苦的弯路。因为这paper作为教材已经非常好了，所以我也不觉得有必要在这份心得里面作什么补充，倒是跟自己总结一下还好。SVD（Singular Value Decomposition）可以把一个表示A-B两个概念的关联性的矩阵（从我的应用角度来说）分解成另一个矩阵表达式（这个表达式有三个矩阵），通过SVD可以加强原来已有的较强的关联性，同时削减原来已有较弱的关联性，具体做法就是在分解后的三个矩阵中，根据特征值矩阵识别哪些列（概念、维度）的特征值足够小，然后裁掉，裁掉这些列后的三个矩阵会变成很小的矩阵，然后重新构造成原来的矩阵（通过矩阵乘法）。我能掌握到的程度目前就只有这些，因为后面肯定还有很重要的一步，就是如何应用SVD重新构造的矩阵（裁剪维度后的分解矩阵）来实现原来要做的事情，这一点是我目前还没掌握的。  

单单掌握SVD的基本原理还不足够，我还需要知道在工程计算程序中一般是如何实现这个算法。于是我又找到了下面这些资料:：[《Matrix Computations》1](http://vdisk.weibo.com/s/hRTZX/1353312532)、[《Matrix Computations》2](http://vdisk.weibo.com/s/ioOGX/1353312888)、[《Optimal sorting algorithms for parallel computers》](http://vdisk.weibo.com/s/hVAHJ/1353312942)来学习串行的SVD，然后是并行的SVD，并行是必须的，因为在我的数据规模下不可能依靠单个节点进行计算，慢死了一头猪！到现在基本上这些原理我都理解了，但经过那份paper的教训，我认为只有当我实现过一个可用的并行SVD算法之后，才能认为自己掌握这些内容，因此为了以后可以更高效地去做这个聚类，下一步就是要做好这个实现。  

基本上可以总结一句，就是Austin给我的paper里面涉及的算法我没有实现，因为自己学不会，但我从里面互相关联的资料学到的知识，导致我马上要提到自己设计的一个可用的聚类算法。  

## 从数学分析出发
其实我自己设计的算法是基于在学习SVD时理解到的这么一个知识：当一个矩阵A(m,n)的代表概念C1与C2的关联性（通过w(x,y)权值表示），权值越大关联性越高，那么这个矩阵与它自身转置相乘得到的对称方阵，就有一个很重要的意思。假如转置相乘后的新矩阵AAT行列都是概念C1，那么AAT(x,y)表示x与y在全部C2元素上的权值分布的接近程度。具体到我的应用场景下，矩阵是A(shopID, userID)，A(s1,u1)表示s1得到u1的喜欢数占u1总喜欢数的百分比（稍后解释为什么这样设计），那么AAT就是(shopID, shopID)，AAT(s1,s2)就表示两家商店s1,s2在所有user中的喜欢数占比分布的相似程度，该值越大，说明这两个商店在所有user中的喜欢数占比分布越相似。这个理论的原理用Austin的话来说挺简单的，就是大数乘大数能得到更大的数，因为在转置相乘的过程中，s1与s2的行向量相乘就是各自在每个用户上的喜欢数占比乘积之和，如果对某个用户，s1的占比跟s2的占比很接近，那么乘起来数就大；不接近，那么乘起来数就比较少。另一个重要的假设就是：*最终目标是通过占比分布把具有共同特征的店铺聚类起来，有一个假设就是，如果两个商店在同一个用户上的小红心占比相近，那就表示这个用户认为这两家商店的宝贝具有他/她喜欢的共同特征。*

但这样还是不精准，注意到占比是指对user所有喜欢数的占比，而不是对商店的，也就是在A中，行向量不是单位向量（加起来之和不等于1），这样就可能导致两个分布不相似的商店，因为其中一个商店在较多用户上的占比都高，因此这两个商店的向量相乘后得出的数值也不低；而另一对商店虽然占比分布相似，但因为两者都占比偏小，所以向量相乘得出的值也不高。如果看到这里，觉得这个结果很正常，是对的，那就remind一下：*算法追求的不是找出占比分布的向量乘积高的商店聚类，而是要找出占比分布相似的商店聚类。*所以说了这么多，要做的只是对(shopID,userID)向量进行单位化就可以了。  

经过单位化和转置相乘之后，我就拥有了一个AAT(shopID,shopID)的矩阵，通过对相关性大小的筛选，我就可以指定需要的相关性范围内的所有商店相关数据（以二元组的形式表达）。在我的算法里面，会把符合相关性范围内的元组提取出来构成一个图，最初我以为只要把这个图的连通分量提取出来就可以得到比较满意的结果，而事实上并非如此。因为目前来说相关性都是基于元组的（shop to shop），而这种相关性是没有传递关系的，简单来说三家商店s1、s2和s3，(s1,s2)和(s1,s3)都在相关性范围内，但(s2,s3)则不一定在相关性范围内，如果简单获取连通分量，那么s2跟s3也会被聚类到一齐。如果这种情况很少那其实还不是什么问题，但事实上这样的传递性问题非常严重。  

为了得到真正可以接受的商店聚类，我需要的不只是连通分量，而是互连度高的连通分量，或者团。
