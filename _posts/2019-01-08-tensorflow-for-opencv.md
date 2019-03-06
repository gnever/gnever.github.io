---
layout: post
title: opencv 调用 TensorFlow 训练好的模型
categories: TensorFlow opencv
description: opencv 调用 TensorFlow 训练好的模型
keywords: TensorFlow opencv
---

[官方教程](https://github.com/opencv/opencv/wiki/TensorFlow-Object-Detection-API)

## 背景
opencv 要使用 TensorFlow object_detection 自定义数据集训练后的模型

## 方法
opencv 提供了 load TensorFlow model 的方法

```python
retval = cv.dnn.readNetFromTensorflow(bufferModel, bufferConfig)
```

- 其中 bufferModel 可以使用 [TensorFlow models](https://github.com/tensorflow/models) object_detection  内的 ```export_inference_graph.py``` 导出

```
python export_inference_graph.py \
--input_type image_tensor \
--pipeline_config_path data/ssd_mobilenet_v1.config  \
--trained_checkpoint_prefix training/model.ckpt-***  \
--output_directory output_inference_graph
```

- bufferConfig 则需要使用 [opencv/samples/dnn](https://github.com/opencv/opencv/tree/master/samples/dnn) 内的 ```tf_text_graph_ssd.py```  导出

``` python
 python tf_text_graph_ssd.py \
--input /path/to/model.pb \
--config /path/to/example.config \
--output /path/to/graph.pbtx 
```

