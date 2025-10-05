---
title: 支持向量机（SVM）初见
date: 2025-05-09 17:15:40
tags: 机器学习
---

# SVM 学习

## 概念
- SVM分为：
    - **SVC**（分类任务）
    - **SVR**（回归任务），股票分析主要用SVR
- **超平面**：k-1维的最佳分类标准
- **支持向量**：离超平面最近的点
- **支持向量机**：找到支持向量，构建超平面
- **核函数**：将低维空间映射到高维空间，使得数据线性可分

## 分析
以 `get_train_model` 函数为例：

```python
def get_train_model(dataset):
    # 设置训练阶树
    DEGREE_POLY = 1
    
    # 进行因子筛选
    selected_factors = get_ic_with_sharpe(dataset)
    
    # 进行训练集和测试集提取
    X = dataset[selected_factors].values
    y = dataset['sharpe'].values
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)
    
    # SVM 的 Pipeline
    svm_pipe = Pipeline([
        ('sc', StandardScaler()), 
        ('poly', PolynomialFeatures(degree=DEGREE_POLY)),
        ('svm', SVR())
    ])  # 添加多项式特征
    
    # 设置 SVM 的参数的取值范围
    svm_params = {
        'svm__C': np.logspace(-2, 3, 6),
        'svm__degree': [1, 2, 3, 4],
        'svm__kernel': ['poly']
    }
    
    # 设置 SVM 的 GridSearchCV 实例
    svm_gs = GridSearchCV(svm_pipe, svm_params, scoring='r2', n_jobs=-1, cv=min(5, len(X_train) - 1))
    
    # 训练 SVM 模型
    svm_gs.fit(X_train, y_train)
    best_svm_pipe = svm_gs.best_estimator_
    best_svm_pipe.fit(X_train, y_train)
    print(best_svm_pipe)
    return best_svm_pipe
```

## 步骤：

### 1. 数据读取和预处理
1. 设置阶数（因为用的核函数是多项式核）
2. 筛选因子
3. 数据集划分，分为训练集和测试集

### 2. 模型构建（使用 Pipeline ）
1. 标准化：统一数据尺度，提升模型性能，避免某些特征因数值范围过大而对模型产生不成比例的影响
2. 多项式特征转换：数据升维
3. SVM回归器（SVR）

### 3. 参数优化
1. 参数网格搜索：排列组合每种参数组合
    * C值：惩罚项，越大，对错误分类的惩罚越重，模型会尽量正确分类所有训练样本，可能导致过拟合
    * 多项式阶数：阶数越高，模型越复杂，越容易过拟合
    * 核函数：将数据映射到更高维空间，使线性不可分的问题变得线性可分
2. 交叉验证：评估每组参数性能，可防止过拟合，（做法：将训练集分成k份···）

### 4. 训练模型
使用最佳参数训练，返回训练最佳模型
