+++
title = "深度神经网络的梯度不稳定问题"
date = "2021-05-06 18:10:00 +0800"
tags = ["Machine Learning"]
categories = ["technology"]
slug = "gradient_instability_problem_of_deep_neural_network"
indent = false
dropCap = false
# katex = true
math = true
+++

> **梯度不稳定问题**根本原因在于前面层上的梯度是来自于后面层上梯度的乘积。当存在过多的层次时，就出现了内在本质上的不稳定场景。


## 梯度消失&爆炸

### 量化分析

可以考虑使用其它激活函数对梯度消失问题进行改善，如ReLU。

> 深度神经网络中的梯度不稳定性，前面层中的梯度或会消失，或会爆炸。无论CNN还是RNN，随着网络深度的加深，梯度消失问题会越愈加明显。此文章讨论了CNN和RNN的梯度问题。  

## 梯度消失

无论CNN还是RNN，随着网络深度的加深，梯度消失问题会越愈加明显。

什么形式会导致梯度爆炸: $f_{t+1}$

累加的形式不消失: $f_{t+1} = w * f_{t} + w f_{t-1} + …$

## CNN

设输入数据为 $x$，对 $x$ 的卷积操作就可以看做是 $Wx+b$

我们设第一层卷积的参数为 $W_1$,$b_1$，第二层卷积的参数是 $W_2$,$b_2$，依次类推。又设激活函数为 $f$ ，每一层卷积在经过激活函数前的值为 $a_i$，经过激活函数后的值为 $f_i$。

按照上面的表示，在CNN中，输入 $x$ ，  
第一层的输出就是 $f_1=f[W_1x+b_1]$，  
第二层的输出就是 $f_2=f[W_2f[W_1x+b_1]+b_2]$，  
第三层的输出就是 $f3=f[W_3f[W_2f[W_1x+b_1]+b_2]+b_3]$。  
设最终损失为 $L$ ，我们来尝试从第三层开始，用BP算法推导一下损失对参数 $W_1$ 的偏导数，看看会发生什么。

为了简洁起见，略过求导过程，最后的结果为

$\frac{∂L} {∂W_1} = \frac {∂L}{∂f_3} \frac{∂f_3}{∂a_3}W_3 \frac{∂f_2}{∂a_2}W_2 \frac{∂f_1}{∂a_1} \frac{∂a_1}{∂W_1}$

**我们常常说原始神经网络的梯度消失问题，这里的 $\frac{∂f_3}{∂a_3}$, $\frac{∂f_2}{∂a_2}$ 就是梯度消失的“罪魁祸首”。**

例如sigmoid的函数，它的导数的取值范围是 $(0, 0.25]$ ，也就是说对于导数中的每一个元素，我们都有  
$$0<\frac{\partial{f_3}}{\partial{a_3}}\le0.25, 0<\frac{\partial{f_2}}{\partial{a_2}}\le0.25$$

小于1的数乘在一起，必然是越乘越小的。这才仅仅是3层，如果10层的话， 根据$0.25^{10}\approx0.000000954$，第10层的误差相对第一层卷积的参数 $W_1$的梯度将是一个非常小的值，这就是所谓的“**梯度消失**”。

ReLU函数的改进就是它使得 $\frac{\partial{f_3}}{\partial{a_3}}\in{0,1}$ ， $\frac{\partial{f_2}}{\partial{a_2}}\in{0,1}$ ， $\frac{\partial{f_1}}{\partial{a_1}}\in{0,1}$ ，这样的话只要一条路径上的导数都是1，无论神经网络是多少层，这一部分的乘积都始终为1，因此深层的梯度也可以传递到浅层中。

> 那为什么同样的方法在RNN中不奏效呢？其实这一点Hinton在它的IRNN论文里面（arxiv：[1504.00941] A Simple Way to Initialize Recurrent Networks of Rectified Linear Units）是很明确的提到的：

也就是说在RNN中直接把激活函数换成ReLU会导致非常大的输出值。为了讲清楚这一点，我们先用同上面相似的符号把原始的RNN表示出来：

在这个表示中，RNN每个阶段的输入是 $x_i$，和CNN每一层使用独立的参数$W_i$不同，原始的
RNN在每个阶段都共享一个参数 $W$。如果我们假设从某一层开始输入 $x_i$ 和偏置 $b_i$ 都为0，
那么最后得到的输出就是 $f[W…[Wf[Wf[Wf_i]]]]$ ，这在某种程度上相当于对参数矩阵 $W$ 作连乘，
很显然，只要 $W$ 有一个大于1的特征值，在经过若干次连乘后都会导致结果是一个数值非常庞大的矩阵。

另外一方面，将激活函数换成ReLU也不能解决梯度在长程上传递的问题。同样考虑 $f_3$ 对 $W$ 的导数。在CNN中，每一层的参数 $W_1,W_2,W_3……$ 是互相独立的，然而RNN中 W 参与了每个时间段的运算，因此 $f_3$ 对 $W$ 导数更复杂，写出来是

$\frac{\partial{f_3}}{\partial{W_1}}=\frac{\partial{f_3}}{\partial{a_3}}f_2+\frac{\partial{f_3}}{\partial{a_3}}W\frac{\partial{f_2}}{\partial{a_2}}f_1+\frac{\partial{f_3}}{\partial{a_3}}W\frac{\partial{f_2}}{\partial{a_2}}W\frac{\partial{f_1}}{\partial{a_1}}\frac{\partial{a_1}}{\partial{W_1}}$ 。我们可以看下最后 $\frac{\partial{f_3}}{\partial{a_3}}W\frac{\partial{f_2}}{\partial{a_2}}W\frac{\partial{f_1}}{\partial{a_1}}\frac{\partial{a_1}}{\partial{W_1}}$

这部分，使用ReLU后，当梯度可以传递时，有
$\frac{\partial{f_3}}{\partial{a_3}}=\frac{\partial{f_2}}{\partial{a_2}}=\frac{\partial{f_3}}{\partial{a_1}}=1$ ，但这个式子中还是会有两个 $W$ 的连乘。在更长程上，就会有更多 $W$ 的连乘。对于CNN来说，这个部分是 $W_1,W_2,W_3…..$ 进行连乘，一方面它们都是稀疏矩阵，另一方面 $W_1,W_2,W_3….$ 互不相同，很大程度上都能抵消掉梯度爆炸的影响。

最后，IRNN在RNN上使用了ReLU，取得了比较好的结果，其中的原因在于，它对 $W$ 和 $b_i$ 取了比较特殊的初值： $W=I$ , $b_i=0$ 。
这样在梯度的式子 $\frac{\partial{f_3}}{\partial{a_3}}W\frac{\partial{f_2}}{\partial{a_2}}W\frac{\partial{f_1}}{\partial{a_1}}\frac{\partial{a_1}}{\partial{W_1}}$ 中 $W$ 尽管会连乘，但是会类似于单位矩阵 $I$ 的连乘，不会引起太明显的梯度数值变化。另外一方面，也不会引起非常大的输出值

## RNN

## LSTM

### 梯度爆炸处理策略

梯度爆炸问题其实不是什么麻烦。

因为现在大家都会做某种形式的 Gradient clipping（也就是限定一下梯度绝对值的上限，超过就截断）来避免梯度爆炸。  
- 觉得 Gradient clipping 很糙？其实一点都不糙，因为用SGD训练深度模型数学上本身就已经糙的不能再糙了。  
- 觉得LSTM不需要这种东西？No。如果查查主流工具包，或者看看比较实际的LSTM应用论文，应该都至少这么做了。

SGD为什么糙？只有GD才不糙？一切近似算法都糙？

> saizheng: relu确实容易explode，除非加大很tricky的clipping，因为clipping多了，优化就做不好了
> 为什么？？
>
> saizheng: 普通rnn做超长memory一般就用quoc le的那个trick，identity init，我猜你看过那个paper，但是那个东西的tradeoff就是损失了短时间段内fit复杂nonlinearty的能力

## 参考

- [RNN中为什么要采用tanh而不是ReLu作为激活函数？](https://www.zhihu.com/question/61265076)