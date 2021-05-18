---
layout: post
title: Pytorch 笔记
subtitle: 
author: ixsluo
date: 2020-11-23 18:40:13 +0800
categories: 
tag: 
---

- GPU设置
```python
# 确定GPU是否可用以及GPU数量
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
n_gpu = torch.cuda.device_count()

net = NET()  # 实例化网络

```

- 尝试单个batch过拟合，确认网络在工作
```python
first_batch = next(iter(train_loader))
for batch_idx, (data) in enumerate([first_batch] * 50):
    # train
```