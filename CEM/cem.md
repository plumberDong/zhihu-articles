#! https://zhuanlan.zhihu.com/p/506155891
# 广义精确匹配的R实现

## 1. 匹配的原因

匹配法在因果统计推断中扮演者越来越显著的作用。因果统计推断的目的是获得处理变量（$T=1, 0$）对因变量$Y$的净效应，而匹配可以使得实验组与对照组之间的混淆变量$X$保持平衡，从而保证效应量估计的无偏性和一致性。

事实上，对混淆变量进行控制的思想在传统线性回归中也有体现，然而，匹配的最大优点，在于避免的模型依赖问题（model dependence）。所谓模型依赖，是指估计量会因模型设置的不同而发生极大变化的情况。一些投机型研究者可以利用模型依赖现象，通过增删控制变量或添加高阶项来一步步“试”出符合发表需要的结果。

相比之下，匹配方法相当于将混淆变量的处理交给了计算机，降低了研究者主观因素的影响，且基于匹配后数据的分析中，估计量受模型设置的影响会很小。


## 2. 广义精确匹配的优点

最常见的匹配方法是倾向值匹配（propensity score matching, PSM）。在该方法中，混淆变量$X$会被降维成一个反映该个体$i$接受实验处理概率的标量$\pi_i$。不同个体间的距离以$\pi_i$差值的绝对值进行反映，用于匹配过程。

然而，学界对PSM方法是存在一定争议的，研究方法大牛[Gary King](https://gking.harvard.edu/files/gking/files/psnot.pdf)曾指出，PSM在一些情境下不仅没有效率，还会因为“修剪”掉过多的样本而导致实验组与对照组平衡性下降。

相比之下，精确匹配（exact matching）方法固然可以达到完美的样本“平衡”，但是当混淆变量$X$过多时，会出现维度灾难（dimension curse），找到精确匹配的个体几乎不可能。此时，广义精确匹配（coarsened exact matching, CEM）的优势就体现出来了。它是一个易于理解、十分高效而且可以缓解维度增长问题的方法。

## 3. CEM的步骤

总体上看，CEM的步骤只有两步：

1. 将所有混淆变量$X$重新编码为分类变量$\tilde X$（对连续型变量而言，会被划分为包含区间的分类变量；对原本就是分类型的变量而言，可以根据自己需要对不同类别进行合并）
2. 基于新生成的$\tilde X$执行精确匹配。

这里以一个简单的案例来说明。

### 3.1 案例说明

假设我们想调查参加某培训（$T=1, 0$）对个体收入（$Y$）的影响，混淆变量为年龄*age*和受教育年限*educ*。下面的代码可以生成一份模拟数据：

```S
library(tidyverse)
set.seed(12345)
# 模拟数据
# 首先是对照组的数据
educ_c <- rbeta(50, shape1 = 2, shape2 = 2)
age_c <- rbeta(50, shape1 = 2, shape2 = 2)
income_c <- educ_c * 20 + age_c * 20 + 10 + rnorm(40, 0, 2)

# 实验组的数据
educ_t <- rbeta(30, shape1 = 2, shape2 = 5)
age_t <- rbeta(30, shape1 = 2, shape2 = 5)
income_t <- educ_t * 20 + age_t * 20 + 20 + rnorm(20, 0, 2)

# 标签: 实验组T，对照组C
labels <- c(rep("C", 50), rep("T", 30))

# 合并为数据框
dat <- tibble(edu = c(educ_c, educ_t),
              age = c(age_c, age_t),
              income = c(income_c, income_t),
              labels = labels)
```

在上面代码中:

- 实验组和对照组样本量分别为$30$和$50$。
- 对照组的年龄和教育水平来自贝塔分布$beta(2,2)$；实验组的年龄和教育水平来自贝塔分布$beta(2,5)$.
- 收入水平为:

$$
Y = 10 + 20 \times 教育水平 + 20 \times 年龄 + 10 \times T + \epsilon \\
\epsilon \sim N(0, 4) \tag{1}
$$

也就是说，真实的处理效应是10。我们可以基于现有数据直接求实验组和对照组的均值差异：

```S
> coef(lm(income ~ labels, data = dat))
(Intercept)     labelsT 
 31.4944798   0.4999942 
```

可以看到，处理效应估计出来只有0.5，相对10的真实效应存在明显的低估。如果观察一下对照组和实验组混淆变量的分布情况，低估的原因就一目了然了：

```S
ggplot(data = dat, aes(x = edu, y = age)) +
  geom_text(aes(label = labels, color = labels)) +
  theme(legend.position='none')
```

![Fig.1](https://pic4.zhimg.com/80/v2-7b8693394c2fb750b2e51e0798d9dc9b.png)

显然，对照组**C**在年龄和教育水平上的平均值更大，而在式1中，这两项的系数都是正值。也就是说，简单均值比较所得的效应量，实际上包含了这些混淆变量的影响。

## 3.2 执行CEM

推荐使用R中的`MathIt`包进行CEM匹配:

```S
library(MatchIt)
m.out <- matchit(labels ~ edu + age, 
                 data = dat, 
                 method = "cem",
                 cutpoints = list(edu =c(.25, .5, .75),
                                  age = c(.25, .5, .75))
                 )
```

- `labels ~ edu + age`设定了处理变量以及我们希望平衡的混淆变量。
- `method = "cem"`设定了匹配方法为CEM
- `cutpoints`以列表的形式设定了我们希望通过哪些分界点对连续变量进行分割。

更详细的参数设定方法可以参阅[官网](https://kosukeimai.github.io/MatchIt/reference/method_cem.html)。

查看一下匹配后的结果：

```S
match.data(m.out) %>%
  ggplot(aes(x = edu, y = age)) +
  geom_text(aes(label = labels, color = labels)) +
  xlim(c(0, 1)) +
  ylim(c(0, 1)) +
  geom_hline(yintercept = c(.25, .5, .75), linetype = "dashed") +
  geom_vline(xintercept = c(.25, .5, .75), linetype = "dashed")
```

![Fig.2](https://pic4.zhimg.com/80/v2-233a783542e02a02203c14076a3c66b8.png)

如上图，如果一个小方块内同时存在对照、实验组个体，这个方块内的所有个体都会被保留到匹配后的数据集中。

## 3.3 权重问题

由于CEM不是一个$1:1$的匹配，要对实验处理组的平均处理效应（average treatment effect on the treated, ATT）进行估计，还必须进行加权调整。

`MatchIt`的加权方法如下：

- 对于实验组**T**，全部赋值为$1$(因为是估计ATT)
- 对于对照组**C**：
  1. 将一个方格（或者说stratum）内的实验组个数除以对照组个数（以`Fig.2`左上角方格为例，共有1个实验组个体，3个对照组个体，那么首先得到$1/3$）
  2. 将所得数值乘上匹配后样本中实验组个数与对照组个数的比值。（再该案例中，匹配后样本有30个来自实验组，23个来自对照组，比值为$23/30$，乘上$1/3$得到权重$23/90=0.2555556$）。

将权重映射到字符大小上会更加直观：

![Fig.3](https://pic4.zhimg.com/80/v2-7d130d17d7d035c935d263bec3523ede.png)

## 3.4 估计

得到匹配样本后，通过简单的t检验就可以对处理效应进行估计了，当然，不要忘记添加权重信息：

```S
> lm(income ~ labels, data = match.data(m.out), weights = weights) %>% summary()

Call:
lm(formula = income ~ labels, data = match.data(m.out), weights = weights)

Weighted Residuals:
    Min      1Q  Median      3Q     Max 
-10.969  -2.675   1.098   4.292   9.383 

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept)   21.922      1.032  21.252  < 2e-16 ***
labelsT       10.073      1.371   7.347 1.54e-09 ***
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

Residual standard error: 4.947 on 51 degrees of freedom
Multiple R-squared:  0.5142,	Adjusted R-squared:  0.5046 
F-statistic: 53.97 on 1 and 51 DF,  p-value: 1.543e-09
```

可以看到，处理效应估计值为$10.073(se = 1.371, p <.001)$，已经非常接近我们设定的真实值10了。