# 🍥 超参数调试、Batch 正则化、Softmax 回归

---

> 🔈 上节我们主要介绍了深度神经网络的优化算法。包括对原始数据集进行分割，使用 mini-batch gradient descent。然后介绍了指数加权平均（Exponentially weighted averages）的概念以及偏移校正（bias correction）方法。接着，我们着重介绍了三种常用的加速神经网络学习速度的三种算法：动量梯度下降、RMSprop 和 Adam 算法。其中，Adam结合了动量梯度下降和RMSprop各自的优点，实际应用中表现更好。然后，我们介绍了另外一种提高学习速度的方法：learning rate decay，通过不断减小学习因子，减小步进长度，来减小梯度振荡。最后，我们对深度学习中local optima的概念作了更深入的解释。本节我们将重点介绍超参数调试和 Batch 正则化。

## 1. 调试处理 Tuning process

深度神经网络需要调试的**超参数（Hyperparameters）**较多，包括：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003095446.png" style="zoom:67%;" />

超参数之间也有重要性差异。**通常来说，学习因子 α  是最重要的超参数，也是需要重点调试的超参数**。动量梯度下降因子 β 、各隐藏层神经元个数 #hidden units 和 mini-batch size 的重要性仅次于 *α*。然后就是神经网络层数 #layers 和学习因子下降参数 learning rate decay。最后，Adam 算法的三个参数 *β*1, *β*2, *ε* 一般常设置为 0.9，0.999 和 $10^{-8}$，不需要反复调试。当然，这里超参数重要性的排名并不是绝对的，具体情况，具体分析。

如何选择和调试超参数？**传统的机器学习中，我们对每个参数等距离选取任意个数的点，然后，分别使用不同点对应的参数组合进行训练，最后根据验证集上的表现好坏，来选定最佳的参数**。例如有两个待调试的参数，分别在每个参数上选取5个点，这样构成了 5×5=25 中参数组合，如下图所示：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003095739.png" style="zoom: 80%;" />

这种做法在参数比较少的时候效果较好。但是**在深度神经网络模型中，我们一般不采用这种均匀间隔取点的方法，比较好的做法是使用随机选择**。也就是说，对于上面这个例子，我们随机选择25个点，作为待调试的超参数，如下图所示：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003095803.png" style="zoom:80%;" />

<u>这种做法带来的另外一个好处就是对重要性不同的参数之间的选择效果更好</u>。假设 hyperparameter1 为 *α*，hyperparameter2 为 ε ，显然二者的重要性是不一样的。如果使用第一种均匀采样的方法，ε 的影响很小，相当于只选择了5个 *α* 值。而如果使用第二种随机采样的方法，ε 和 *α* 都有可能选择25种不同值。这大大增加了 *α* 调试的个数，更有可能选择到最优值。其实，在实际应用中完全不知道哪个参数更加重要的情况下，随机采样的方式能有效解决这一问题，但是均匀采样做不到这点。

在经过随机采样之后，我们可能得到某些区域模型的表现较好。然而，为了得到更精确的最佳参数，我们应该继续对选定的区域进行由粗到细的采样（coarse to fine sampling scheme）。也就是**放大表现较好的区域，再对此区域做更密集的随机采样**。例如，对下图中右下角的方形区域再做25点的随机采样，以获得最佳参数：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003100053.png" style="zoom:80%;" />



## 2. 为超参数选择合适的范围appropriate scale

上一部分讲的调试参数使用随机采样，**对于某些超参数是可以进行尺度均匀采样的，但是某些超参数需要选择不同的合适尺度进行随机采样**。

什么意思呢？例如对于超参数 #layers 和 #hidden units，都是正整数，是可以进行**均匀随机采样**的，**即超参数每次变化的尺度都是一致的（如每次变化为1，犹如一个刻度尺一样，刻度是均匀的）**。

但是，对于某些超参数，可能需要**非均匀随机采样（即非均匀刻度尺）**。例如超参数 *α*，待调范围是 [0.0001, 1]。如果使用均匀随机采样，那么有90%的采样点分布在[0.1, 1]之间，只有10%分布在[0.0001, 0.1]之间。这在实际应用中是不太好的，最佳的 *α* 值可能主要分布在[0.0001, 0.1]之间，因此我们更关注的是区间 [0.0001, 0.1]，应该在这个区间内细分更多刻度。

**通常的做法是将 linear scale 转换为 log scale，将均匀尺度转化为非均匀尺度，然后再在 log scale 下进行均匀采样**。这样，[0.0001, 0.001]，[0.001 , 0.01]，[0.01, 0.1]，[0.1, 1]各个区间内随机采样的超参数个数基本一致，也就扩大了之前[0.0001, 0.1]区间内采样值个数。

一般解法是，如果线性区间为 [a, b]，令 m=log(a)，n=log(b)，则对应的 log 区间为 [m,n]。对 log 区间的 [m,n] 进行随机均匀采样，然后得到的采样值 r，最后反推到线性区间，即 $10^r$。$10^r$就是最终采样的超参数：

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003101159.png)

相应的 Python 语句为：

```python
m = np.log10(a)
n = np.log10(b)
r = np.random.rand()
r = m + (n-m)*r
r = np.power(10,r)
```

**除了 *α* 之外，动量梯度因子 *β* 也是一样，在超参数调试的时候也需要进行非均匀采样**。一般 *β* 的取值范围在[0.9, 0.999]之间，那么1−*β* 的取值范围就在[0.001, 0.1]之间。那么直接对1−*β*在[0.001, 0.1]区间内进行log变换即可。

这里解释下为什么 *β* 也需要向 *α*  那样做非均匀采样。假设 *β* 从0.9000变化为0.9005，那么 $\frac{1}{1-\beta}$ 基本没有变化。但假设 *β* 从0.9990变化为0.9995，那么 $\frac{1}{1-\beta}$ 前后差别1000。 ***β* 越接近1，指数加权平均的个数越多，变化越大。所以对 *β* 接近 1 的区间，应该采集得更密集一些**。

## 3. 超参数调试的实践：Pandas VS Caviar 

经过调试选择完最佳的超参数并不是一成不变的，一段时间之后（例如一个月），需要根据新的数据和实际情况，再次调试超参数，以获得实时的最佳模型。

在训练深度神经网络时，一种情况是受计算能力所限，我们只能对一个模型进行训练，调试不同的超参数，使得这个模型有最佳的表现。我们称之为 **Babysitting one model**。另外一种情况是可以对多个模型同时进行训练，每个模型上调试不同的超参数，根据表现情况，选择最佳的模型。我们称之为 **Training many models in parallel**。

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003103456.png)

因为第一种情况只使用一个模型，所以类比做 **Panda approach**（**熊猫方式**。当熊猫有了孩子，他们的孩子非常少，一次通常只有一个，然后他们花费很多精力抚养熊猫宝宝以确保其能成活）

第二种情况同时训练多个模型，类比做 **Caviar approach**（**鱼子酱方式**。在交配季节，有些鱼类会产下一亿颗卵，但鱼类繁殖的方式是，它们会产生很多卵，但不对其中任何一个多加照料，只是希望其中一个，或其中一群，能够表现出色）

使用哪种模型是由计算资源、计算能力所决定的。一般来说，对于非常复杂或者数据量很大的模型，使用 Panda approach 更多一些。

## 4. 隐藏层的标准化处理 Batch Normalization

Sergey Ioffe 和 Christian Szegedy 两位学者提出了 Batch Normalization 方法。**Batch Normalization** 不仅可以让调试超参数更加简单，而且可以让神经网络模型更加“健壮”。也就是说较好模型可接受的超参数范围更大一些，包容性更强，使得更容易去训练一个深度神经网络。接下来，我们就来介绍什么是 Batch Normalization，以及它是如何工作的。

之前，我们提到过在训练神经网络时，标准化输入可以提高训练的速度。方法是对训练数据集进行归一化的操作，即将原始数据减去其均值 *μ* 后 ，再除以其方差 $\sigma^2$。但是标准化输入只是对输入进行了处理，那么**对于神经网络，又该如何对各隐藏层的输入进行标准化处理呢？**

其实在神经网络中，第 $l$ 层隐藏层的输入就是第 $l -1$ 层隐藏层的输出$ A^{[l-1]}$。对 $A^{[l-1]}$ 进行标准化处理，从原理上来说可以提高 $W^{[l]}$ 和 $b^{[l]}$ 的训练速度和准确度。这种对各隐藏层的标准化处理就是 Batch Normalization。值得注意的是，实际应用中，**一般是对 $Z^{[l-1]}$ 进行标准化处理而不是 $A^{[l-1]}$** ，其实差别不是很大。

Batch Normalization 对第 $l$ 层隐藏层的输入 $Z^{[l-1]}$ 做如下标准化处理，忽略上标 $[l-1]$ ：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003105909.png" style="zoom:80%;" />

其中，**m 是单个 mini-batch 包含样本个数**， ε 是为了防止分母为零，可取值 $10^{-8}$。这样，使得该隐藏层的所有输入 $z^{(i)}$ 均值为 0，方差为 1。

但是，大部分情况下并不希望所有的 $z^{(i)}$ 均值都为0，方差都为1，也不太合理。通常需要对 $z^{(i)}$ 进行进一步处理：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003111239.png" style="zoom:67%;" />

上式中，$\gamma$ 和 $\beta$ 是 learnable parameters，类似于 W 和 b 一样，可以通过梯度下降等算法求得。这里，$\gamma$ **和 $\beta$ 的作用是让 $\tilde z^{(i)}$ 的均值和方差为任意值**，只需调整其值就可以了。例如，令：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003111911.png" style="zoom:67%;" />

则 $\tilde z^{(i)}=z^{(i)}$，即identity function。可见，设置 $\gamma$ 和 $\beta$ 为不同的值，可以得到任意的均值和方差。

这样，通过 Batch Normalization，对隐藏层的各个 $z^{[l](i)}$ 进行标准化处理，得到 $\tilde z^{[l](i)}$，替代 $z^{[l](i)}$。

值得注意的是，**输入的标准化处理 Normalizing inputs 和隐藏层的标准化处理 Batch Normalization 是有区别的**。Normalizing inputs 使所有输入的均值为 0，方差为1。而 Batch Normalization 可使各隐藏层输入的均值和方差为任意值。实际上，从激活函数的角度来说，如果各隐藏层的输入均值在靠近0的区域即处于激活函数的线性区域，这样不利于训练好的非线性神经网络，得到的模型效果也不会太好。这也解释了为什么需要用 $\gamma$ 和 $\beta$ 来对 $z^{[l](i)}$ 作进一步处理。

## 5. 将 Batch Norm 拟合进神经网络

我们已经知道了如何对某单一隐藏层的所有神经元进行 Batch Norm，接下来将研究如何把 Bath Norm 应用到整个神经网络中。

对于 L 层神经网络，经过 Batch Norm 的作用，整体流程如下：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003113355.png" style="zoom: 80%;" />

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003114102.png" style="zoom: 50%;" />

## 6. Batch Norm 为什么奏效

我们可以把输入特征做均值为0，方差为1的规范化处理，来加快学习速度。而Batch Norm也是对隐藏层各神经元的输入做类似的规范化处理。总的来说，**Batch Norm不仅能够提高神经网络训练速度，而且能让神经网络的权重W的更新更加“稳健”**，尤其在深层神经网络中更加明显。比如神经网络很后面的W对前面的W包容性更强，**即前面的W的变化对后面W造成的影响很小，整体网络更加健壮**。

举个例子来说明，假如用一个浅层神经网络（类似逻辑回归）来训练识别猫的模型。如下图所示，提供的训练样本只有黑猫，而测试集拥有各种颜色的猫。测试的结果可能并不好。其原因是**训练样本不具有一般性**（即不是所有的猫都是黑猫），这种训练样本（黑猫）和测试样本（猫）分布的变化称之为 **Covariate shift**。

![](https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003114525.png)

对于这种情况，如果实际应用的样本与训练样本分布不同，即发生了Covariate shift，则一般是要对模型重新进行训练的。在神经网络，尤其是深度神经网络中，Covariate shift 会导致模型预测效果变差，重新训练的模型各隐藏层的 $W^{[l]}$ 和 $B^{[l]}$ 均产生偏移、变化。

而 **Batch Norm 的作用恰恰是减小 Covariate shift 的影响，让模型变得更加健壮，鲁棒性更强**。**Batch Norm减少了各层 $W^{[l]}$ 、$B^{[l]}$ 之间的耦合性，让各层更加独立，实现自我训练学习的效果**。也就是说，如果输入发生Covariate shift，那么因为Batch Norm的作用，对个隐藏层输出 $Z^{[l]}$ 进行均值和方差的归一化处理，$W^{[l]}$ 和 $B^{[l]}$ 更加稳定，使得原来的模型也有不错的表现。针对上面这个黑猫的例子，如果我们使用深层神经网络，使用 Batch Norm，那么该模型对花猫的识别能力应该也是不错的。

从另一个方面来说，Batch Norm也起到轻微的正则化（regularization）效果。具体表现在：

- 每个 mini-batch 都进行均值为0，方差为1的归一化操作
- **每个 mini-batch中，对各个隐藏层的 $Z^{[l]}$ 添加了随机噪声，效果类似于 Dropout**
- **mini-batch越小，正则化效果越明显**

但是，Batch Norm的正则化效果比较微弱，正则化也不是Batch Norm的主要功能。

## 7. 测试时的 Batch Norm

训练过程中，Batch Norm是对单个mini-batch进行操作的，但在测试过程中，如果是单个样本，该如何使用Batch Norm进行处理呢？

首先，回顾一下训练过程中Batch Norm的主要过程：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003115044.png" style="zoom: 50%;" />

<u>其中， μ 和 $\sigma^2$ 是对单个mini-batch中所有m个样本求得的。在测试过程中，如果只有一个样本，求其均值和方差是没有意义的，就需要对 μ 和 $\sigma^2$ 进行估计</u>。

估计的方法有很多，理论上我们可以将所有训练集放入最终的神经网络模型中，然后将每个隐藏层计算得到的 $\mu^{[l]}$ 和 $\sigma^{2[l]}$ 直接作为测试过程的 $\mu$ 和 $\sigma^2$ 来使用。但是，实际应用中一般不使用这种方法，而是使用我们之前介绍过的**指数加权平均**（exponentially weighted average）的方法来预测测试过程单个样本的 μ 和$ \sigma^2$。

指数加权平均的做法很简单，对于第 l 层隐藏层，考虑所有 mini-batch 在该隐藏层下的 $\mu^{[l]}$ 和 $\sigma^{2[l]}$ ，然后用指数加权平均的方式来预测得到当前单个样本的 $\mu^{[l]}$ 和 $\sigma^{2[l]}$ 。这样就实现了对测试过程单个样本的均值和方差估计。最后，再利用训练过程得到的 γ 和 *β* 值计算出各层的 $\tilde z^{(i)}$ 值。

## 8. Softmax 回归

目前我们介绍的都是二分类问题，神经网络输出层只有一个神经元，表示预测输出 $\hat y$ 是正类的概率 $P(y=1|x)$，$\hat y >0.5$ 则判断为正类，$\hat y < 0.5$ 则判断为负类。

对于多分类问题，用C表示种类个数，神经网络中输出层就有C个神经元，即 $n^{[L]}=C$ 。其中，每个神经元的输出依次对应属于该类的概率，即 $P(y=c|x)$ 。**为了处理多分类问题，我们一般使用 Softmax 回归模型**。Softmax 回归模型**输出层**的激活函数（**Softmax 激活函数**）如下所示：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003145635.png" style="zoom: 50%;" />

输出层每个神经元的输出 $a^{[L]}_i$ 对应属于该类的概率，满足：<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003145702.png" style="zoom:50%;" />

所有的 $a^{[L]}_i$ ，即 $\hat y$ ，维度为 (C, 1)。

🚨 **Softmax 激活函数应用于多分类问题的<u>输出层</u>，将多个神经元的输出映射到（0,1）区间内，可以看成是当前输出是属于各个分类的概率，从而来进行多分类。**

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003152055.png" style="zoom: 80%;" />

> 💡 之前，我们的激活函数都是接受单行数值输入，例如 **Sigmoid** 和 **ReLu** 激活函数，输入一个**实数**，输出一个实数。**Softmax** 激活函数的特殊之处在于，因为需要将所有可能的输出归一化，就需要输入一个**向量**，最后输出一个向量。

下面给出几个简单的线性多分类的例子：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003145803.png" style="zoom: 80%;" />

如果使用神经网络，特别是深层神经网络，可以得到更复杂、更精确的非线性模型。

## 9. 训练一个 Softmax 分类器

Softmax classifier 的训练过程与我们之前介绍的二元分类问题有所不同。

先来看一下 **loss function**。举例来说，假如 C=4，某个样本的预测输出 $\hat y$ 和真实输出 *y* 为：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003151030.png"  />

从 $\hat y$ 值来看，$P(y=4|x)=0.4$ ，概率最大，而真实样本属于第 2 类，因此该预测效果不佳。我们定义 Softmax classifier 的 loss function 为：

⭐ <img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003151141.png" style="zoom:50%;" />

然而，由于只有当 $j=2$ 时，$y_2=1$ ，其它情况下，$y_j=0$ 。所以，上式中的 $L(\hat y,y)$ 可以简化为：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003151254.png" style="zoom:50%;" />

要让 $L(\hat y,y)$ 更小，就应该让 $\hat y_2$ 越大越好。$\hat y_2$ 反映的是概率，完全符合我们之前的定义。

所有 m 个样本的 **cost function** 为：

⭐ <img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003151414.png" style="zoom:50%;" />

其预测输出向量 $A^{[L]}$ 即 $\hat Y$ 的维度为(4, m)。

Softmax classifier 的反向传播过程仍然使用梯度下降算法，其推导过程与二元分类有一点点不一样。因为**只有输出层的激活函数不一样**，我们先推导 $dZ^{[L]}$ ：

<img src="https://cs-wiki.oss-cn-shanghai.aliyuncs.com/img/20201003151539.png" style="zoom:52%;" />

可见 $dZ^{[L]}$ 的表达式与二元分类结果是一致的，虽然推导过程不太一样。然后就可以继续进行反向传播过程的梯度下降算法了，推导过程与二元分类神经网络完全一致。


## 📚 Reference

- [黄海广 - Coursera 深度学习教程中文笔记](https://github.com/fengdu78/deeplearning_ai_books)
- [红色石头 - 吴恩达 deeplearning.ai 专项课程精炼笔记](https://redstonewill.com/category/ai-notes/andrew-deeplearning-ai/page/2/)
- [学习笔记6：激活函数之 Softmax](https://blog.csdn.net/softdiamonds/article/details/80101440)