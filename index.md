---
layout: post
title: "Survival Analysis Report - Project 1"
date: 2026-04-27
---

# Q2 (2) Survival Analysis Report

## 1. 题目概述
本题要求按照官方 survival analysis casebook 的流程，在服务器上完成生存分析案例复现，并在报告中说明整个分析过程及主要结果。该案例使用 IBM 的 Telco Customer Churn 数据集，将 **客户流失（churn）** 视为事件，将 **tenure** 视为生存时间，从而研究客户在不同条件下的流失风险和留存概率。官方教程的整体主线包括：先构建 Bronze/Silver 数据，再依次完成 Kaplan-Meier、Cox Proportional Hazards、Accelerated Failure Time 分析，最后将生存模型的输出用于 Customer Lifetime Value 场景。

## 2. 数据准备过程
我首先按照教程思路读取原始 CSV 文件，并手动定义 schema，然后使用 Spark DataFrame 构造原始层数据。接着，我对原始数据进行了筛选和转换，构造后续建模使用的 Silver 数据集。

具体来说，主要做了三步处理：
* **第一**：将原来的 `churnString` 转换为数值型的 `churn` 事件变量；
* **第二**：仅保留 `contract = 'Month-to-month'` 的客户；
* **第三**：去掉 `internetService = 'No'` 的客户。

教程将这个经过清洗后的数据集称为 silver table，后续所有生存分析都建立在这份数据之上。在完成 Silver 数据集构造之后，我进一步将其转换为 Pandas DataFrame，并交给 `lifelines` 库完成生存分析模型拟合。这个处理方式与官方教程保持一致：前半部分使用 Spark 进行数据准备，后半部分使用 Pandas + lifelines 进行 Kaplan-Meier、Cox PH 和 AFT 建模。

## 3. Kaplan-Meier 生存分析
在 Kaplan-Meier 部分，我使用 `tenure` 作为生存时间，使用 `churn` 作为事件变量，拟合总体客户群体的生存曲线。根据教程输出，模型共使用 3351 个样本，其中 1795 个为 right-censored 样本，这说明有相当一部分客户在观察期结束前并未发生流失事件，因此使用 survival analysis 是合理的。

总体生存曲线反映了客户在不同时间点仍然留存的概率。教程中给出的中位生存时间为 **34.0 个月**，这意味着客户的生存概率下降到 0.5 大约发生在第 34 个月左右。这个结果可以作为对整体客户留存情况的一个直观概括。

在协变量层面，我进一步比较了不同分组的生存曲线，并结合 log-rank test 进行统计检验。教程中特别提到：
* `gender` 的两条生存曲线比较接近，因此其区分能力较弱；
* `onlineSecurity` 的组间曲线差异更明显，因此更可能与 churn 风险相关。

此外，Kaplan-Meier 部分还提取了 `internetService = DSL` 这一子群体在不同时间点的 survival probability。Kaplan-Meier 的输出不仅可以用于可视化，也可以作为后续业务分析或下游模型的输入。

## 4. Cox Proportional Hazards 模型
在完成 Kaplan-Meier 分析后，我继续进行 Cox Proportional Hazards 多变量建模。与 Kaplan-Meier 不同，Cox PH 可以同时分析多个协变量对 hazard ratio 的影响。

拟合结果显示，模型中各个协变量的 p-value 均低于 0.005，具有统计显著性。以 `internetService_DSL` 为例，其系数为 -0.22，exp(coef) 为 0.80，说明与基准组相比，DSL 用户的流失风险更低。

不过，教程通过统计检验、Schoenfeld residuals 和 log-log Kaplan-Meier plots 检查了 **proportional hazards assumption（比例风险假设）**。检验结果表明，该模型在多个变量上（如 `internetService_DSL`、`onlineBackup_Yes` 等）违反了该假设。这意味着如果目标是严格的统计推断，还需要考虑进一步的模型改进。

## 5. Accelerated Failure Time 模型
AFT 与前两者的最大不同在于，它属于参数模型，需要为生存时间指定具体分布。教程中使用的是 **Log-Logistic AFT**。

拟合结果显示，AFT 模型基于 3351 个样本完成，中位生存时间的指数化结果为 135.51。教程指出，AFT 模型结果中各协变量同样具有较高统计显著性。通过 log-odds 风格的诊断图判断：图中各组线条整体上比较直，说明 log-logistic 分布是相对合理的；但这些线条并不十分平行，说明 AFT 对这个特定模型来说也并不是完全理想的。

## 6. Customer Lifetime Value 应用
在最后一个部分，教程展示了如何将 survival model 的输出应用到 **Customer Lifetime Value（CLV）** 分析中。

核心思想是：先使用 Cox PH 模型预测某一客户画像在未来各个月份的 survival probability，再结合每月利润、折现率等参数，计算期望月利润、折现后的净现值（NPV）以及累计净现值（Cumulative NPV）。

教程在 Databricks 中生成了 `payback_df`，主要包括：
* **Survival Probability**
* **Avg Expected Monthly Profit**（survival probability 乘以该客户方案的月利润）
* **NPV**（考虑资金时间价值后的折现值）
* **Cumulative NPV**（累计净现值）

这说明 survival analysis 不只是统计模型，还可以进一步服务于企业的客户获取、预算分配和生命周期价值估计。

## 7. 总结
通过本题，我完整复现了官方 survival analysis casebook 的主要流程。整体来看，Kaplan-Meier 更适合做单变量的留存比较，Cox PH 更适合分析多协变量的共同影响，而 AFT 提供了一个参数化的替代思路。同时，本案例也说明：模型拟合结果本身并不代表完全可靠，必须结合假设检验与诊断图进行综合判断。

这个案例不仅帮助我理解了 survival analysis 在客户流失分析中的应用，也展示了如何将模型输出转化为业务价值。
