---
layout: default
title: 机器学习----从回归谈起
categories: [机器学习]
---
<center><font size="14"> 机器学习----从回归谈起</font></center>
从这篇文章开始进入机器学习。
从一个后端程序员的角度来看机器学习，重点要弄清楚三个问题：  
1. 学什么？  
2. 怎么用？  
1. 怎么学？

让我们从周志华的《机器学习》 一书的习题3.3 开始：

<img src='/static/image/ml/watermelon_dataset_3.0_alpha.png' /> 
<p/>
现在要求实现一个算法，用来判断一个西瓜是不是“好瓜“？  
通过观察数据，大概可以猜到西瓜的好坏大概跟密度和含糖率相关。如果要求写一个C++函数用来判断西瓜是否是好瓜，大概会是这样的
```c++
bool JugeWatermelon(float density,float sugarContent){
    bool bRet= 某一个算法过程
    return bRet;
}
```
###  从这里开始我们进入了机器学习的世界。
##### 首先要先选择一个模型。因为是学习所以直接给定是逻辑回归（logistic regression）
这个模型长这样：

>
$$
\begin{aligned}
z &= \textcolor{red}{密度系数}*密度值 + \textcolor{red}{含糖率系数} * 含糖值 + \textcolor{red}{误差修正} \\
好瓜&= \begin{Bmatrix}
   否，z \leqslant 0\\
   是，z > 0
\end{Bmatrix}
\end{aligned}
$$

红字部分就是机器学习需要学习的内容！
### 这里可以回答第一个问题：“机器学习”学什么？
通过“学习”，可以得知：
> 
* 密度系数 = -20.09168508
* 密度值 = 102.31669719
* 误差修正 = -8.43271713

当确定要一个模型以后，机器学习要把模型所需要的系数“学习”出来，也就是计算出来。
但是机器学习**并不学习模型本身！！！**


### 怎么用？
这个其实就很简单了，将这些系数填入模型里面就可以了。
```C++
bool JugeWatermelon(float density,float sugarContent){
    float z =-20.09168508 * density + 102.31669719 * sugarContent - 8.43271713;
    bool bRet= false;
    if (z > 0){
        bRet = true;
    }
    return bRet;
}
```

这样就可以通过JugeWatermelon来判断西瓜的好坏了。

可以写一段代码来验证一下：
```c++
#include <vector>
using namespace std;
bool JugeWatermelon(float density, float sugarContent) {
    float z = -20.09168508 * density + 102.31669719 * sugarContent - 8.43271713;
    int ret = 0;
    if (z > 0) {
        ret = 1;
    }
    return ret;
}
int main() {
    vector<vector<float>> data = { {1,  0.697, 0.46,   1},
                                  {2,  0.774, 0.376,  1},
                                  {3,  0.634, 0.264,  1},
                                  {4,  0.608, 0.318,  1},
                                  {5,  0.556, 0.215,  1},
                                  {6,  0.403, 0.237,  1},
                                  {7,  0.481, 0.149,  1},
                                  {8,  0.437, 0.211,  1},
                                  {9,  0.666, 0.091,  0},
                                  {10, 0.243, 0.0267, 0},
                                  {11, 0.245, 0.057,  0},
                                  {12, 0.343, 0.099,  0},
                                  {13, 0.639, 0.161,  0},
                                  {14, 0.657, 0.198,  0},
                                  {15, 0.36,  0.37,   0},
                                  {16, 0.593, 0.042,  0},
                                  {17, 0.719, 0.103,  0}};


    //int bRet = JugeWatermelon()
    int right = 0;
    int error = 0;
    for (vector<vector<float>>::iterator itData = data.begin();
         itData != data.end(); ++itData) {
        const int  index = static_cast<int>((*itData)[0]);
        const float density = (*itData)[1];
        const float sugarContent = (*itData)[2];
        const int gootJugeWater = static_cast<int>((*itData)[3]);
        const int pred = JugeWatermelon(density, sugarContent);
        const char* resultTxt = "";
        if (pred == 1){
            resultTxt = "好瓜";
        }else {
           resultTxt = "孬瓜";
        }

        printf("第%d个西瓜，预测结果是：%s ",index,resultTxt);
        if (gootJugeWater == pred){
            right++;
            printf("预测正确\n");
        } else {
            error++;
            printf("**预测错误**\n");
        }
    }
    printf("一共预测了%d个西瓜，预测正确：%d个，错误：%d个。正确率：%f",data.size(),right,error,float(right)/data.size());
    return 1;
```
运行结果是：
>第1个西瓜，预测结果是：好瓜 预测正确  
>第2个西瓜，预测结果是：好瓜 预测正确  
>第3个西瓜，预测结果是：好瓜 预测正确  
>第4个西瓜，预测结果是：好瓜 预测正确  
>第5个西瓜，预测结果是：好瓜 预测正确  
>第6个西瓜，预测结果是：好瓜 预测正确  
>第7个西瓜，预测结果是：孬瓜 \*\*预测错误\*\*  
>第8个西瓜，预测结果是：好瓜 预测正确  
>第9个西瓜，预测结果是：孬瓜 预测正确  
>第10个西瓜，预测结果是：孬瓜 预测正确  
>第11个西瓜，预测结果是：孬瓜 预测正确  
>第12个西瓜，预测结果是：孬瓜 预测正确  
>第13个西瓜，预测结果是：孬瓜 预测正确  
>第14个西瓜，预测结果是：孬瓜 预测正确  
>第15个西瓜，预测结果是：好瓜 \*\*预测错误\*\*  
>第16个西瓜，预测结果是：孬瓜 预测正确  
>第17个西瓜，预测结果是：孬瓜 预测正确  
>一共预测了17个西瓜，预测正确：15个，错误：2个。正确率：0.882353

###  怎么度量结果？
正确率接近90%， ***<font color="#ADADAD">还是挺高的</font>***。    
这个***<font color="#ADADAD">还是挺高的</font>***说的好没有底气啊。    
专业的说法是要引入**二分类代价矩阵**， 又叫**混淆矩阵(Confusion Matrix)**。    
  

<img src='/static/image/ml/2019-11-15-confusionMatrix.png' /> 

查准率:

$$
\begin{aligned}
P &= { {TP} \over {TP+FP} } \\
  &={ {8} \over{8+1} }\\
  & \approx 0.89
\end{aligned}
$$

查全率：

$$
\begin{aligned}
P &= { {TP} \over {TP+FN} } \\
  &={ {8} \over{8+1} }\\
  & \approx 0.89
\end{aligned}
$$

F1度量= 
$$
\begin{aligned}
P &= { {2 \times P \times R } \over {P+ R} } \\
  &={ {2 \times 0.89 \times 0.89 } \over{0.89+0.89} }\\
  & = 0.89
\end{aligned}
$$

### 接下来到了最后一个问题，怎么学？
根据《机器学习》一书的公式3.27

$$
\ell (\beta) = \sum_{i=1}^m (-y_i\beta^T \hat x_i + ln(1+e^{\beta^T\hat x_i}))
$$

采用梯度下降法来求解
$$
\beta^* =  arg \space min \space \ell(\beta)
$$

参考CSDN博客上文章的python实现：
https://blog.csdn.net/snoopy_yuan/article/details/63684219

``` python
def likelihood_sub(x, y, beta):
    '''
    @param x: one sample variables
    @param y: one sample label
    @param beta: the parameter vector in 3.27
    @return: the sub_log-likelihood of 3.27
    '''
    return -y * np.dot(beta, x.T) + np.math.log(1 + np.math.exp(np.dot(beta, x.T)))

def likelihood(X, y, beta):
    '''
    @param X: the sample variables matrix
    @param y: the sample label matrix
    @param beta: the parameter vector in 3.27
    @return: the log-likelihood of 3.27
    '''
    sum = 0
    m,n = np.shape(X)

    for i in range(m):
        sum += likelihood_sub(X[i], y[i], beta)

    return sum
```
 基于训练集（注意x->[x,1]），给出基于3.27似然函数的定步长梯度下降法，注意这里的偏梯度实现技巧：
``` python
'''
@param X: X is the variable matrix
@param y: y is the label array
@return: the best parameter estimate of 3.27
'''
def gradDscent_1(X, y):  #implementation of basic gradDscent algorithms

    h = 0.1  # step length of iteration
    max_times= 500  # give the iterative times limit
    m, n = np.shape(X)

    beta = np.zeros(n)  # parameter and initial to 0
    delta_beta = np.ones(n)*h  # delta parameter and initial to h
    llh = 0
    llh_temp = 0

    for i in range(max_times):
        beta_temp = beta.copy()

        # for partial derivative
        for j in range(n):
            beta[j] += delta_beta[j]
            llh_tmp = likelihood(X, y, beta)
            delta_beta[j] = -h * (llh_tmp - llh) / delta_beta[j]
            beta[j] = beta_temp[j]

        beta += delta_beta
        llh = likelihood(X, y, beta)

    return beta
```
