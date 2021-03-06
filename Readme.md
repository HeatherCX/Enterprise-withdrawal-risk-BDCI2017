# 队伍组成

队伍名称：DataNTL(Data Never tells a Lie)

队伍组成：Evander  HeatherCX 邹雨恒 张奋涛 李莱  

# 赛题回顾

[本赛题](http://www.datafountain.cn/#/competitions/271/intro)以企业为中心，围绕企业主体在多方面留下的行为足迹信息构建训练数据集，以企业在未来两年内是否因经营不善退出市场作为目标变量进行预测。参赛者需要利用训练数据集中企业信息数据，构建算法模型，并利用该算法模型对验证数据集中企业，给出预测结果以及风险概率值。预测结果以AUC值作为主要评估标准，在AUC值（保留到小数点后3位数字）相同的情况下，则以F1-score（命中率及覆盖率）进行辅助评估。

数据集下载：[下载链接](https://pan.baidu.com/s/1pLzbwfx)（BDCI2017-liangzi.rar为初赛数据，BDCI2017-liangzi-Semi.zip为复赛数据，初赛数据与复赛数据中有一些数据格式不一致，详情见***********.md）

# 解决方案概述

本赛题初赛给出了***个企业，企业基本信息数据（1entbase.csv）、变更数据（2alter.csv）、分支机构数据（3branch.csv）、投资数据（4invest.csv）、权利数据（5right.csv）、项目数据（6project.csv）、被执行数据（7lawsuit.csv）、失信数据（8breakfaith.csv）、招聘数据（9recruit.csv）。我们依照每个表的数据特征和信息，在每一个表内构建特征，然后利用xgboost、gbdt、dart等模型对数据进行训练和预测。

复赛中新增了数据：企业资质10qualification.csv

# 特征工程

##1entbase.csv

1. RGYEAR 成立年度GAP
2. HY 行业大类 one-hot编码
3. ETYPE 企业类型 one-hot编码
4. 注册资本和各种身份指标 'ZCZB', 'MPNUM', 'INUM', 'ENUM','FINZB', 'FSTINUM', 'TZINUM'
5. 注册资本onehot编码
6. FINZB/ZCZB   FINZB+ZCZB
7. ZCZB用KMeans聚类
8. 省份类别
9. PROV独热编码
10. EID数值

## 2alter.csv

1. 企业变更次数统计
2. 变更事项代码one-hot编码
3. 2013-2015年变更次数的统计/2013-2015年变更次数趋势
4. ALTAF/ALTBE比值
5. 美元、港币、人民币分类
6. 最早和最后一次更变的时间
7. alter_year - rgyear
8. ALTAF-ALTBE

## 3branch.csv

1. 分支企业数量统计
2. 分支机构在省内的数量和比例
3. 分支成立年度到2017的GAP
4. 分支成立年度到关停时的GAP
5. 分支survive的count和rate
6. 分支关停的count和rate
7. 分支成立趋势
8. b_reyear-rgyear
9. branch b_endyear-rgyear

## 4invest.csv

1. 投资企业数量统计
2. 投资企业在省内的数量统计(空值赋0.5),投资企业在省内的比例统计
3. 投资企业存活的个数/比例
4. 投资企业成立趋势
5. 持股总数
6. 平均持股比例
7. 被投资企业类型
8. BTYEAR - rgyear

##5right.csv

1. 企业专利count
2. 权利类型ONE-HOT编码
3. 申请日期GAP
4. 权利申请到赋予的时间GAP
5. right_typecode数字
6. right_typecode不同类型权利个数
7. right_date最早和最后时间
8. right_year - rgyear

## 6project.csv

1. project_count项目数量统计
2. project_timegap
3. DJDATE_2013/2014/2015
4. ifhome-count//ifhome_ratio
5. last year
6. 每年事件数发生趋势
7. TYPECODE(最小&最大)
8. 同项目的eid是否继续经营的和sametype_target_sum

## 7lawsuit.csv

1. month gap
2. lawamount
3. lawamount-zczb
4. 事件数
5. 每年被执行金额//趋势//year
6. TYPECODE

## 8breakfaith.csv

1. 失信最早年份
2. SXENDDATE
3. 每年发生次数
4. month-gap
5. TYPECODE
6. breakfaith - rgyear

## 9recruit.csv

1. recruit_RECRNUM_count 招聘总次数
2. recruit_RECRNUM_sum 招聘职位总数[重要]
3. recurt_info_ration平均每次招聘的职位数量[重要]
4. WZCODE_ZP01~ZP03:招聘WZCODE的one-hot
5. month_gap:最近招聘的月度gap
6. PNUM
7. SUM_PNUM
8. recruit - rgyear

## 10qualification.csv

1. addtype
2. 资质数量
3.   

# 模型设计和融合

基于以上提取到的特征，进行模型设计与融合

- 单模型

  我们采用的单模型有xgboost、gbdt(lgb)、dart(lgb)，其中在初赛阶段，xgboost的效果最好，gbdt与dart效果相近。模型训练速度的话，lgb的速度要快于xgboost，gbdt的速度要快于dart的速度。

- 加权融合

  加权最开始我们采用的是直接平均的方法，即$(Score(xgb)+Score(gbdt)+Score(dart))/3$

  然后我们尝试利用Stacking的思想来确定不同模型的权重，将不同模型的Score作为Stacking第二层的输入，stacking第二层的模型采用线性模型，通过对训练集不同模型Score的训练，来确定模型的权重

- Stacking

  我们的Stacking的第一层模型采用了两种方法

  1. 对于同一类模型（eg. gbdt），通过设定不同的seed或者是random state，来随机产生不同的模型
  2. 利用不同的模型（eg. xgboost、gbdt、dart）对数据进行训练和预测，得到每个模型的输出作为第二层stacking的输入

# 文档说明

- extract_feature：划分数据集，提取特征，生成训练集和预测集
- model：训练xgboost、gbdt、dart模型，生成特征重要性文件，生成预测结果。

# 环境说明

Python3

Package：Numpy, Pandas, sklearn, hyperopt, xgboost 和 lightgbm



