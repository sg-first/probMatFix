有条件情况下的多目标分类修正：基于概率矩阵变换的方法
=============

问题介绍
------------
### 多目标分类是啥？
举个目标检测的例子：目标检测的过程是，在图上画n个框，并且推断这n个框里的东西是哪个分布，得出n个分布。然后我可以对每个分布argmax一下，选择概率最大的结果，然后就可以输出这个n个框里是啥东西了

### 有条件是啥意思？
上面的过程看着挺好的，但是假如我事先知道，这个图里只有三个人两个车，我是否可以用这个先验增强先前推断的结果？先前我只是对单个分布argmax，但是没有考虑其它分布的情况。比如说一共五个框，四个框的argmax结果是人，概率分别为90% 90% 90% 60%，本来我的推断结果应该是有四个人，但我事先知道这个图里只有三个人，那我的结果里是否应该只令前三个是人，排除最后一个？

注意，我的目标是让整体准确率最高，而不是必须选择三个框猜人，剩下两个框猜车。比如四个人argmax框的概率全部为90%，我没法判断它们谁的概率更高，只知道它们是人（而不是车）的概率都很大，那我的理性选择应该是这四个全猜人，尽管里面会有猜错的，但我掌握的信息只能做出这个选择

### 那么，我们面对的问题描述如下：
有N个随机变量（n个框），它们分别对应N个分布（n次分类的softmax结果），我们要猜测这N个随机变量的真实值

我们有先验信息（三个人两个车）。如何同时考虑先验和这N个分布，然后猜出更准确的结果？

解法
--------
四个框的argmax结果都是人，我该怎么选呢？需要注意的是，我们不能片面地比较`P(第1个框是人)`、`P(第2个框是人)`、`P(第3个框是人)`、`P(第4个框是人)`、的概率然后选top3（前面已经举过反例了，我们要让整体准确率最高，片面地只选三个是行不通的）。我们要最大化**猜对的数量的期望**

目标检测这个情景说起来有点绕，下面我们给该问题换成一个等价的表述：猜选择题

### 怎样计算期望
举个栗子：
```
    A   B       C
1   0.9 0.05    0.05
2   0.8 0.11    0.09
3   0.7 0.15    0.15
```
#### 计算每种选择的期望
假设我1、2两道题都选了A，它们为正确答案的概率是0.9、0.8。我要计算它们正确个数的期望。这类似于二项分布，但区别是，每次试验的成功率不同，因此我们要遍历所有排列而非组合。对于每种排列，我们可以计算联合概率：
``` python
thisProb = 第1题为A/不为A的概率 * 第2题为A/不为A的概率 * ……  第n题为A/不为A的概率
#“/”是“或”，不是除号。对于每个选项，每道题有正确错误两种情况，因此将先产生2^n种排列
```
所有排列的联合概率之和构成分母，基于先验我们可以知道，一共有`maxTimes`道题选A。我们排除所有正确数量大于`maxTimes`的排列（它们不被加进分母里）。正确个数为`k``(k=0-maxTimes)`的联合概率之和构成n个分子。二者都得到后，我们可以计算分布列，随后计算出数学期望

#### 基于先验的期望下界
先验还给期望bound了最小值。比如先验地知道`一共有3道题选A的有1个`，如果我`3道题全选A`那么一定能猜对这一个；如果先验地知道`一共有3道题选A的有2个`，如果我`任选两道题选A`，那么至少能猜对一个。因此这里面是有规律的：我尽量让`先验选的范围`和`我选的范围`重合的最少，最后还是重合的部分，就是`至少猜对的数量`。因此公式如下：
```
Exp_min = 我的选择覆盖的部分 - 先验没覆盖的部分 = len(我的选择数量) - (len(总数) - len(先验选择的数量))
```

#### merge result
因此，对于每个选项，我们要分别计算`选该选项的题`的`期望（不考虑先验）`和`期望最小值（考虑先验）`，然后取max。对各选项都走这个步骤计算，把它们的结果加和即可

理论完整解法
-----------------
本来我是想想一个根据先验修正概率值（即第一步算出的期望）的方法。理论上的方法应该是基于贝叶斯定理（全概率公式），比如某道题选A的情况：
```
P(后验选A) = P(我选A)*P(先验选A) / (P(我选A)*P(先验选A) + P(我不选A)*P(先验不选A))
```
但需要注意的是，`P(先验选A)`是与状态相关的。比如你知道20题中有5题选A，那当你选第一题的时候，`P(第一题先验选A)=5/20`，如果你第一题选了A，那么下一道题仍然选A的概率就会有：
```
P(第二题先验选A)=(5-P(第一题后验选A))/20
```
如果你第一题没选A，那么：
```
P(第二题先验选A)=P(第一题先验选A)
```
因此，在每道题上的概率是与先前做的所有选择有关的。因此想要求最大后验，需要在所有排列中选取