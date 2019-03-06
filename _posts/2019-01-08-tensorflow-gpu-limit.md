---
layout: post
title: 约束 TensorFlow 只在 GPU 上运行
categories: TensorFlow
description: 约束 TensorFlow 只在 GPU 上运行
keywords: TensorFlow
---

显卡资源有限，当有显卡被占用时可以让 TensorFlow 在  CPU 上继续执行任务

只需要添加如下代码即可

```
import os
os.environ["CUDA_DEVICE_ORDER"] = "PCI_BUS_ID"  
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"
```

该方法是让 tf 无法调用 cuda，降级到使用 CPU 资源进行计算
