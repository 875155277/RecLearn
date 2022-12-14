<div>
  <img src='https://cdn.jsdelivr.net/gh/BlackSpaceGZY/cdn/img/logo.jpg' width='36%'/>
</div>

## RecLearn

<p align="left">
  <img src='https://img.shields.io/badge/python-3.8+-blue'>
  <img src='https://img.shields.io/badge/Tensorflow-2.5+-blue'>
  <img src='https://img.shields.io/badge/License-MIT-blue'>
  <img src='https://img.shields.io/badge/NumPy-1.17-brightgreen'>
  <img src='https://img.shields.io/badge/pandas-1.0.5-brightgreen'>
  <img src='https://img.shields.io/badge/sklearn-0.23.2-brightgreen'>
</p> 

[简体中文](https://github.com/ZiyaoGeng/Recommender-System-with-TF2.0/blob/reclearn/README_CN.md) | [English](https://github.com/ZiyaoGeng/Recommender-System-with-TF2.0/tree/reclearn)

RecLearn（Recommender Learning）对`Recommender System with TF2.0`中 [master](https://github.com/ZiyaoGeng/RecLearn/tree/master) 分支的内容进行了归纳、整理，是一个基于Python和Tensorflow2.x开发的推荐学习框架，适合学生、初学者研究使用。**当然如果你更习惯master分支中的内容，并希望对其中的内容进行修改、更新，可以直接clone整个包的内容进行使用**。实现的推荐算法按照工业界的两个应用阶段进行分类：

- matching recommendation stage
- ranking  recommendeation stage



## 更新

**23/04.2022**：更新了所有的召回模型。



## 安装

### Package

RecLearn已经上传在pypi上，可以使用`pip`进行安装：

```shell
pip install reclearn
```

所依赖的环境：

- python3.8+
- Tensorflow2.5-GPU+/Tensorflow2.5-CPU+
- sklearn0.23+

### Local

也可以直接clone Reclearn到本地：

```shell
git clone -b reclearn git@github.com:ZiyaoGeng/RecLearn.git
```



## 快速开始

在`example`中，给出了每一个推荐模型的demo。

### Matching

**1、划分数据集**

给定数据集的路径：

```python
file_path = 'data/ml-1m/ratings.dat'
```

划分当前数据集为训练集、验证集、测试集。如果你使用了`movielens-1m`、`Amazon-Beauty`、`Amazon-Games`、`STEAM`数据集的话，也可以直接调用Reclearn中`data/datasets/*`的方法，完成划分：

```python
train_path, val_path, test_path, meta_path = ml.split_seq_data(file_path=file_path)
```

其中`meta_path`为元文件的路径，元文件保存了用户、物品索引的最大值。

**2、加载数据**

完成对训练集、验证集、测试集的读取，并且对每一个正样本分别生成若干个负样本（随即采样），数据的格式为字典：

```
data = {'pos_item':, 'neg_item': , ['user': , 'click_seq': ,...]}
```

如果你构建的模型为序列推荐模型，需要引入点击序列。对于上述4个数据集，Reclearn提供了加载数据的方法：

```python
# general recommendation model
train_data = ml.load_data(train_path, neg_num, max_item_num)
# sequence recommendation model, and use the user feature.
train_data = ml.load_seq_data(train_path, "train", seq_len, neg_num, max_item_num, contain_user=True)
```

**3、给定超参数**

模型需要指定所需的超参数，以`BPR`模型为例：

```python
model_params = {
        'user_num': max_user_num + 1,
        'item_num': max_item_num + 1,
        'embed_dim': FLAGS.embed_dim,
        'use_l2norm': FLAGS.use_l2norm,
        'embed_reg': FLAGS.embed_reg
    }
```

**4、构建模型、编译**

选择或构建你需要的模型，并进行编译。以`BPR`为例：

```python
model = BPR(**model_params)
model.compile(optimizer=Adam(learning_rate=FLAGS.learning_rate))
```

如果你对模型的结构存在问题的话，编译之后可以调用`summary`方法打印查看：

```python
model.summary()
```

**5、学习以及预测**。

```python
for epoch in range(1, epochs + 1):
    t1 = time()
    model.fit(
        x=train_data,
        epochs=1,
        validation_data=val_data,
        batch_size=batch_size
    )
    t2 = time()
    eval_dict = eval_pos_neg(model, test_data, ['hr', 'mrr', 'ndcg'], k, batch_size)
    print('Iteration %d Fit [%.1f s], Evaluate [%.1f s]: HR = %.4f, MRR = %.4f, NDCG = %.4f'
          % (epoch, t2 - t1, time() - t2, eval_dict['hr'], eval_dict['mrr'], eval_dict['ndcg']))
```

### Ranking

针对Criteo数据集，采用了两种数据处理方法：加载部分数据训练模型或者通过分割数据集的方法使用全部数据训练。第一种方法参考`example/train_small_criteo_demo.py`。第二种方法参考`example/r_deepfm_demo.py`文件，具体如下所示：

**1、分割数据集**

调用`reclearn.data.datasets.criteo.get_split_file_path(parent_path, dataset_path, sample_num)`方法可以将数据集分割，`sample_num`确定每一个子集样本数量，所以子集保存在数据集对应的路径。若之前已经分割完成，没有改变子数据集路径可以直接读取，或者可以赋值`parent_path`。

```python
sample_num = 4600000
    split_file_list = get_split_file_path(dataset_path=file, sample_num=sample_num)
```

**2、建立特征映射**

分割数据集后，在整个数据集下对所有的特征进行映射（静态Embedding层需要确定大小），并且密集数据类型进行分桶处理转化为离散数据类型。调用`get_fea_map(fea_map_path, split_file_list)`方法，最后保存为映射文件保存为`fea_map.pkl`。若之前已经完成该步骤，可以赋值`fea_map_path`参数。

```python
# If you want to make feature map.
fea_map = get_fea_map(split_file_list=split_file_list)
# Or if you want to load feature map.
# fea_map = get_fea_map(fea_map_path='data/criteo/split/fea_map.pkl')
```

**3、加载测试集**

选择最后一个子数据集作为测试集。

```python
feature_columns, test_data = create_criteo_dataset(split_file_list[-1], fea_map)
```

**4、构建模型**

```python
model = FM(feature_columns=feature_columns, **model_params)
model.summary()
model.compile(loss=binary_crossentropy, optimizer=Adam(learning_rate=learning_rate),
              metrics=[AUC()])
```

**5、迭代训练，并验证**

```python
for file in split_file_list[:-1]:
    print("load %s" % file)
    _, train_data = create_criteo_dataset(file, fea_map)
    # TODO: Fit
    model.fit(
        x=train_data[0],
        y=train_data[1],
        epochs=1,
        batch_size=batch_size,
        validation_split=0.1
    )
    # TODO: Test
    print('test AUC: %f' % model.evaluate(x=test_data[0], y=test_data[1], batch_size=batch_size)[1])
```



## 实验结果

Reclearn所设计的实验环境与部分论文不同，所以结果可能会存在一定偏差，具体请参考[experiement](./docs/experiment.md)。

### Matching

<table style="text-align:center;margin:auto">
  <tr></tr>
  <tr>
    <th rowspan="2">Model</th>
    <th colspan="3">ml-1m</th>
    <th colspan="3">Beauty</th>
    <th colspan="3">STEAM</th>
  </tr>
  <tr>
    <th>HR@10</th><th>MRR@10</th><th>NDCG@10</th>
    <th>HR@10</th><th>MRR@10</th><th>NDCG@10</th>
    <th>HR@10</th><th>MRR@10</th><th>NDCG@10</th>
  </tr>
  <tr><td>BPR</td><td>0.5768</td><td>0.2392</td><td>0.3016</td><td>0.3708</td><td>0.2108</td><td>0.2485</td><td>0.7728</td><td>0.4220</td><td>0.5054</td></tr>
  <tr><td>NCF</td><td>0.5834</td><td>0.2219</td><td>0.3060</td><td>0.5448</td><td>0.2831</td><td>0.3451</td><td>0.7768</td><td>0.4273</td><td>0.5103</td></tr>
  <tr><td>DSSM</td><td>0.5498</td><td>0.2148</td><td>0.2929</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr>
  <tr><td>YoutubeDNN</td><td>0.6737</td><td>0.3414</td><td>0.4201</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td></tr>
  <tr><td>GRU4Rec</td><td>0.7969</td><td>0.4698</td><td>0.5483</td><td>0.5211</td><td>0.2724</td><td>0.3312</td><td>0.8501</td><td>0.5486</td><td>0.6209</td></tr>
  <tr><td>Caser</td><td>0.7916</td><td>0.4450</td><td>0.5280</td><td>0.5487</td><td>0.2884</td><td>0.3501</td><td>0.8275</td><td>0.5064</td><td>0.5832</td></tr>
  <tr><td>SASRec</td><td>0.8103</td><td>0.4812</td><td>0.5605</td><td>0.5230</td><td>0.2781</td><td>0.3355</td><td>0.8606</td><td>0.5669</td><td>0.6374</td></tr>
  <tr><td>AttRec</td><td>0.7873</td><td>0.4578</td><td>0.5363</td><td>0.4995</td><td>0.2695</td><td>0.3229</td><td>-</td><td>-</td><td>-</td></tr>
  <tr><td>FISSA</td><td>0.8106</td><td>0.4953</td><td>0.5713</td><td>0.5431</td><td>0.2851</td><td>0.3462</td><td>0.8635</td><td>0.5682</td><td>0.6391</td></tr>
</table>



### Ranking

<table style="text-align:center;margin:auto">
  <tr></tr>
  <tr>
    <th rowspan="2">Model</th>
    <th colspan="2">500w(Criteo)</th>
    <th colspan="2">Criteo</th>
  </tr>
  <tr>
    <th>Log Loss</th>
    <th>AUC</th>
    <th>Log Loss</th>
    <th>AUC</th>
  </tr>
  <tr><td>FM</td><td>0.4765</td><td>0.7783</td><td>0.4762</td><td>0.7875</td></tr>
  <tr><td>FFM</td><td>-</td><td>-</td><td>-</td><td>-</td></tr>
  <tr><td>WDL</td><td>0.4684</td><td>0.7822</td><td>0.4692</td><td>0.7930</td></tr>
  <tr><td>Deep Crossing</td><td>0.4670</td><td>0.7826</td><td>0.4693</td><td>0.7935</td></tr>
  <tr><td>PNN</td><td>-</td><td>0.7847</td><td>-</td><td>-</td></tr>
  <tr><td>DCN</td><td>-</td><td>0.7823</td><td>0.4691</td><td>0.7929</td></tr>
  <tr><td>NFM</td><td>0.4773</td><td>0.7762</td><td>0.4723</td><td>0.7889</td></tr>
  <tr><td>AFM</td><td>0.4819</td><td>0.7808</td><td>0.4692</td><td>0.7871</td></tr>
  <tr><td>DeepFM</td><td>-</td><td>0.7828</td><td>0.4650</td><td>0.8007</td></tr>
  <tr><td>xDeepFM</td><td>0.4690</td><td>0.7839</td><td>0.4696</td><td>0.7919</td></tr>
</table>


## 复现论文列表

### 召回模型（Top-K推荐）

|                         Paper\|Model                         |  Published   |     Author     |
| :----------------------------------------------------------: | :----------: | :------------: |
| BPR: Bayesian Personalized Ranking from Implicit Feedback\|**MF-BPR** |  UAI, 2009   | Steﬀen Rendle  |
|    Neural network-based Collaborative Filtering\|**NCF**     |  WWW, 2017   |  Xiangnan He   |
| Learning Deep Structured Semantic Models for Web Search using Clickthrough Data\|**DSSM** |  CIKM, 2013  |  Po-Sen Huang  |
| Deep Neural Networks for YouTube Recommendations\| **YoutubeDNN** | RecSys, 2016 | Paul Covington |
| Session-based Recommendations with Recurrent Neural Networks\|**GUR4Rec** |  ICLR, 2016  | Balázs Hidasi  |
|     Self-Attentive Sequential Recommendation\|**SASRec**     |  ICDM, 2018  |      UCSD      |
| Personalized Top-N Sequential Recommendation via Convolutional Sequence Embedding\|**Caser** |  WSDM, 2018  |   Jiaxi Tang   |
| Next Item Recommendation with Self-Attentive Metric Learning\|**AttRec** | AAAAI, 2019  |  Shuai Zhang   |
| FISSA: Fusing Item Similarity Models with Self-Attention Networks for Sequential Recommendation\|**FISSA** | RecSys, 2020 |    Jing Lin    |



### 排序模型（CTR预估）

|                         Paper｜Model                         |  Published   |                            Author                            |
| :----------------------------------------------------------: | :----------: | :----------------------------------------------------------: |
|                Factorization Machines\|**FM**                |  ICDM, 2010  |                        Steffen Rendle                        |
| Field-aware Factorization Machines for CTR Prediction｜**FFM** | RecSys, 2016 |                       Criteo Research                        |
|    Wide & Deep Learning for Recommender Systems｜**WDL**     |  DLRS, 2016  |                         Google Inc.                          |
| Deep Crossing: Web-Scale Modeling without Manually Crafted Combinatorial Features\|**Deep Crossing** |  KDD, 2016   |                      Microsoft Research                      |
| Product-based Neural Networks for User Response Prediction\|**PNN** |  ICDM, 2016  |                Shanghai Jiao Tong University                 |
|    Deep & Cross Network for Ad Click Predictions｜**DCN**    | ADKDD, 2017  |               Stanford University｜Google Inc.               |
| Neural Factorization Machines for Sparse Predictive Analytics\|**NFM** | SIGIR, 2017  |                         Xiangnan He                          |
| Attentional Factorization Machines: Learning the Weight of Feature Interactions via Attention Networks\|**AFM** | IJCAI, 2017  |    Zhejiang University\|National University of Singapore     |
| DeepFM: A Factorization-Machine based Neural Network for CTR Prediction\|**DeepFM** | IJCAI, 2017  | Harbin Institute of Technology\|Noah’s Ark Research Lab, Huawei |
| xDeepFM: Combining Explicit and Implicit Feature Interactions for Recommender Systems\|**xDeepFM** |  KDD, 2018   |        University of Science and Technology of China         |
| Deep Interest Network for Click-Through Rate Prediction\|**DIN** |  KDD, 2018   |                        Alibaba Group                         |



## 讨论

对于项目有任何建议或问题，可以在`Issue`留言。