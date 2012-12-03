---
layout: default
published: true
---

# 合并两个.a文件(static library)
今天呆牛突然问了个问题，怎么把一个.a文件编译到另一个.a文件中。其实在此之前我都已经100%忘记了.a是什么文件，只记得是跟linux C代码有关系的概念。然后马上Google了一下，原来是个static library文件，一般创建这种文件就是先：  
g++ -c file1.cpp  
生成一个fil1.o文件，然后：  
ar -rvs f1.a file1.o  
生成一个f1.a文件，这个f1.a文件里面就会包括一个__.SYMDEF SORTED文件和一个file1.o文件。需要用到这个静态库里面定义的组件时，就这样：
g++ -o main main.cpp f1.a  

现在就是正题，如果两个f1.a和f2.a文件，想把其中的内容合并到一齐，那该怎么做。ar命令里面有一个-r或者-q选项都可以把新的文件追加到一个目标.a文件里面，因此执行：
ar -r f1.a f2.a  
会直接把f2.a加入到f1.a里面去，但这样是用不了f2.a里面的代码的（可能有办法能用到，但我不知道）。既然简单的路走不通，就用个傻傻的方法做了这个事情，思路就是把一个.a文件里面的组件都提取出来，然后逐个加入到目标.a文件里面去。具体命令就是：  
for f in `ar -t f2.a | grep -v "__.SYMDEF SORTED"`; do ar -x f2.a $f; ar -r f1.a $f; rm $f; done;

