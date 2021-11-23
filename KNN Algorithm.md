### KNN Algorithm

#### Data Normalization

归一化是一个让权重变为统一的过程

Definition：归一化就是把需要处理的数据经过处理后（通过某种算法）限制在你需要的一定范围内。首先归一化是为了后面数据处理的方便，其次是保证程序运行时收敛加快。

Methods:

1. Linear Function transformation: 

   $$
   y=\frac{(x-minVal)}{(maxVal-minVal)}
   $$
   

2. Logrithm function transformation:
   $$
   y=log_{10}x
   $$
   

3. Arctangible function transformation:
   $$
   y=\frac{arctan(x)*2}\pi
   $$
   

在统计学中，归一化的具体作用是归纳统一样本的统计分布性。归一化是在0-1之间是统计的概率分布，归一化在-1-+1之间是统计的坐标分布。

#### KNN算法伪代码

```
对于每一个在数据集中的数据点：
	计算目标的数据点（需要分类的数据点）与该数据点的距离
	将距离排序，从小到大
	选取前K个最短距离
	选取这K个钟最多的分类类别
	返回该类别作为目标数据点的预测值
```

#### 基本原理

通过距离度量来计算查询点（query point）与每个训练数据点的距离，然后选出与查询点相近的K个最临点（k nearest neighbours），使用分类决策来选出对应的标签来作为该查询点的标签

#### KNN三要素

1. K，K的取值
2. 距离度量 Metric/Distance Measure
3. 分类决策 Decision rule
