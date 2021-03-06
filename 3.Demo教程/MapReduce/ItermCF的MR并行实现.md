## **ItermCF的基本思想**

基于物品相似度的协同过滤推荐的思想大致可分为两部分：

> 1.计算物与物之前的相似度   
> 2.根据用户的行为历史，给出和历史列表中的物品相似度最高的推荐

**通俗的来讲就是：**

对于物品 A,根据所有用户的历史偏好,喜欢物品 A 的用户都 喜欢物品 C,得出物品 A 和物品 C 比较相似,而用户 C 喜欢物品 A,那么可以推断出 用户 C 可能也喜欢物品 C。

## **ItermCF的算法实现思路**

对于以下的数据集：   
|UserId|ItermId|Preference|
|---|---|---|
|1|	101|	5|
|1|	102|	3|
|1|	103|	2.5
|2|	101|	2|
|2|	102|	2.5|
|2|	103|	5|
|2|	104|	2|
|3|	101|	2|
|3|	104|	4|
|3|	105|	4.5|
|3|	107|	5|
|4|	101|	5|
|4|	103|	3|
|4|	104|	4.5|
|4|	106|	4|
|5|	101|	4|
|5|	102|	3|
|5|	103|	2|
|5|	104|	4|
|5|	105|	3.5|
|5|	106|	4|
|6|	102|	4|
|6|	103|	2|
|6|	105|	3.5|
|6|	107|	4|

### **用户评分矩阵**

首先可以建立用户对物品的评分矩阵，大概长这个样子：

$$
  \begin{matrix}
   & 1 & 2 & 3 & 4 & 5\\
   101 & 5 & 2 & 2 & 5 & 4\\
   102 & 3 & 2.5 & 0 & 0 & 3 \\
   103 & 2.5 & 5 & 0 & 3 & 2\\
   104 & 0 & 2 & 4 & 4.5 & 4\\
   105 & 0 & 0 & 4.5 & 0 & 3\\
   106 & 0 & 0 & 0 & 4 & 4\\
   107 & 0 & 0 & 5 & 4 & 0\\
  \end{matrix} 
$$

列为UserId，行为ItermId，矩阵中的值代表该用户对该物品的评分。   

从列的方向看，该矩阵的每一个列在mr程序中可以用一行简单的字符串来表示：   
```
1	101:5,102:3,103:2.5 ...   
```   
这样一来，上面的矩阵5个列就可以由5行类似的字符串来构成。   
那么第一个mr任务的功能就是一个简单的数据转换过程：

> 1.输入的key为行偏移量，value为每行内容，形如：1,101,5.0
> 2.在map阶段，分割每行内容，输出的key为1，value为101:5.0
> 3.在reduce阶段，将UserId相同的所有评分记录进行汇总拼接，输出的key仍然为1，value形如：101:5,102:3,103:2.5 ...

如此一来通过第一个mr任务得到了用户的评分矩阵。

### **物品同现矩阵**

该矩阵大概长这个样子：

矩阵的值表示，两个物品同时被用户喜欢（评过分）的次数，例如：101和102这个组合被1，2，5三个用户喜欢过，那么在矩阵中101和102对应的值就是3。   

这个矩阵的意义就是**各个物品之间的相似度**，为什么可以这么说？   
如果两个物品经常同时被很多用户喜欢，那么可以说这两个物品是相似的，**同时被越多的用户喜欢（即为通同现度，上面矩阵中的值），这两个物品的相似度就越高**。   
其实观察可以发现，行和列上相同的（比如101和101）相比其他值（比如101和102，101和103）都是最大的，**因为101和101就是同一个物品，相似度肯定是最大的**。   

从列的方向上看，这个同现矩阵的每一列在mr程序中可以通过下面简单的字符串来表示：

```
101:101	5	
101:102	3
101:103	4
...
```

m*n的同现矩阵就由m个以上的字符串（n行）组成。   

那么第二个mr任务的功能就是在第一个mr任务的输出结果上得到物品同现矩阵：

> 1.输入的key为偏移量，输入的value为UserId+制表符+ItermId1:Perference1,ItermId2:Perference2...
> 2.输入的value中，UserId和Perference是不需要关心的，观察物品的同现矩阵，map阶段的工作就是将每行包含的ItermId都解析出来，**全排列组合作为key输出**，每个key的value记为1。
> 3.在reduce阶段所做的就是根据key对value进行累加输出。

如此一来便能够得到物品的同现矩阵。   

### **物品同现矩阵和用户评分矩阵的相乘**

物品同现矩阵*用户评分矩阵=推荐结果：

为什么两个矩阵相乘可以得到推荐结果？   
**物品1和各个物品的同现度*用户对各个物品的喜好度，反应出用户对物品1的喜好度。**   

例如，要预测用户3对103物品的喜好度，我们需要找到和103相似的物品，比如101物品，和103的同现度为4，**是很类似的物品，用户3对101的评分为2，那么一定程度上可以反映出用户对103的喜好度**，101和103的相似度（即同现度）*用户3对101的评分可以得到**用户3对103的喜好度权重**，将用户3对各个物品的权重相加，可以**反映出用户3对103的喜好度**。   

了解矩阵相乘的意义之后，第三个mr任务的功能就是实现两个矩阵的相乘，并将结果输出。

在这个mr任务中，这两个矩阵的相乘可以这样来计算：

将同现矩阵存入一个Map中，形如：

```
Map<String, Map<String, Double>> colItermOccurrenceMap = new HashMap<String, Map<String, Double>>();
```

同现矩阵中的每一行就是大Map中的一条记录，每行对应的每列都在该记录的小Map中。

在map阶段的开始的时候初始化这个Map，输入的value形如101:101 5,101:102 3，将101作为大Map的key，value为小Map，小Map的key为101/102，value为5/3。

由于map函数读取文件是并发读取的，不能保证两个输入文件的读取顺序（**在同一个文件中也不能保证**），所以这里使用Hadoop提供的**分布式缓存机制**来对同现矩阵进行共享。

关于Hadoop的分布式缓存机制请看：    
[Hadoop的DistributedCache机制][2]

初始化同现矩阵之后，读取评分矩阵的每一行，输入的value为1	101:5,102:3,103:2.5 ...    
将每行的itermIds和对应的评分数提取出来，遍历itermId，根据itermId到itermOccurrenceMap中找到对应的List集合，找到每个itermId在该集合中对应的itermId2记录，将评分数*同现度，之后进行累加，以UserId:ItermID作为key，累加值作为value输出。

reduce的工作就很简单了，根据key对value进行累加输出即可。

### **项目代码**

[源码Github地址][3]

作者：[@小黑][1]

[1]:http://www.xiaohei.info
[2]:http://blog.csdn.net/qq1010885678/article/details/50751007
[3]:https://github.com/chubbyjiang/MapReduce