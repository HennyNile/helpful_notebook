# Problems encountered in Model Training 

这里记录一下目前（23.8.1）训练TCNN遇到的问题。

## 1. 目前TCNN的训练结果

### a. 包含两个rels的4023个训练样本上训练的TCNN模型

![tcnn_prediction_two_rels](../../../DBGroup/1.Learning_based_Optimizer/pg_optimizer_experiment_utils/papers/Cost_Model/TCNN/tcnn_prediction_two_rels.png)

### b. 包含三个rels的9000个训练样本上训练的TCNN模型

![image-20230801214419115](C:/Users/2/AppData/Roaming/Typora/typora-user-images/image-20230801214419115.png)

## 2. 目前的问题

### a. 计划的执行时间分布不均匀

![image-20230801220055251](C:/Users/2/AppData/Roaming/Typora/typora-user-images/image-20230801220055251.png)

​                                                                                                  图1: 包含两个关系表的训练样本的执行时间的分布



![image-20230801220349904](C:/Users/2/AppData/Roaming/Typora/typora-user-images/image-20230801220349904.png)

​                                                                                                 图2: 包含三个关系表的训练样本的执行时间的分布

图1和图2分别为包含两个和三个关系表的训练样本的执行时间的分布，横轴是执行时间，纵轴是训练样本的数量。可以观察到大多数的查询的执行时间都小于50s，但是观看训练结果，**两个模型对于执行时间小于50s的样本训练结果没有很好**，但是tile的时间预测的还比较不错。我的想法是：**需不需要强化对执行时间小于50s的样本的预测结果，但这可能会导致tile的时间预测变差**。

### b. 少表数据训练的模型预测多表数据效果很差

标题中的少表数据指包含两个表的训练样本，多表数据指包含三个表的训练样本。由于我们的训练数据最多只包含三个表，因此我有一个想法是：**使包含较少的表的数据（1-3个）训练的模型能够预测包含较多的表的数据（4个以及更多**。因此，我用两个表的数据训练出来的模型预测了一下三个表的数据，结果如下：

![image-20230801221454894](C:/Users/2/AppData/Roaming/Typora/typora-user-images/image-20230801221454894.png)



这个问题仍需要进一步分析解决。

