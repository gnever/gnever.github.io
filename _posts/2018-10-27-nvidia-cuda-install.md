---
layout: post
title: NVIDIA 显卡驱动及CUDA、cuDNN安装
categories: TensorFlow
description: Nvidia 显卡驱动及CUDA、cuDNN各版本的下载安装
keywords: CUDA cuDNN 安装 
---

TensorFlow的GPU版本需要硬件环境的支持,首先要有个NVIDIA显卡，然后确保安装了正确的显卡驱动、CUDA和cuDNN对应版本。
NVIDIA 官网首页提供的都是最新的驱动，但最新版本不一定适用于开发环境，所以要进行版本的选择,下面依次介绍

# NVIDIA显卡驱动安装

[官方驱动下载](https://www.geforce.cn/drivers){:target="_blank"}

## 选择型号选择自己的驱动

我的机器配置是Ubuntu 16.04.4 LTS 64位，显卡 GTX 1060

所以选择如下


![选择适合自己的驱动](/images/posts/nvidia_driver/choose_driver.png)

## 选择合适的版本
刚开始安装的是最新的410.66，但是在开发过程中会有报错，最终选择了 384.13

![选择适合自己的驱动](/images/posts/nvidia_driver/select_driver_version.png)

## 安装

``` shell
chmod + libcudnn7_7.3.0.29-1+cuda9.0_amd64.deb
./libcudnn7_7.3.0.29-1+cuda9.0_amd64.deb
```

# CUDA安装

官方推荐 CUDA 9 + cuDNN7，那么版本选择上就按照这个来了

## 下载

首页推荐的CUDA 10，所以找到 [CUDA历史版本](https://developer.nvidia.com/cuda-toolkit-archive){:target="_blank"}


选择 [CUDA Toolkit 9.0 ](https://developer.nvidia.com/cuda-90-download-archive){:target="_blank"}


按照提示选择合适自己的下载文件

这里使用了 deb(local) 的类型

![选择适合自己的文件类型](/images/posts/nvidia_driver/cuda_select.png)

## 安装

``` shell
sudo dpkg -i cuda-repo-ubuntu1604-9-0-local_9.0.176-1_amd64.deb
sudo apt-key add /var/cuda-repo-<version>/7fa2af80.pub
sudo apt-get update
sudo apt-get install cuda
```

# cuDNN 安装
[cyDNN历史版本](https://developer.nvidia.com/rdp/cudnn-archive){:target="_blank"}


[cuDNN v7.3.0 Runtime Library for Ubuntu16.04 (Deb)](https://developer.nvidia.com/compute/machine-learning/cudnn/secure/v7.3.0/prod/9.0_2018920/Ubuntu16_04-x64/libcudnn7_7.3.0.29-1-cuda9.0_amd64){:target="_blank"}


![选择适合自己的文件类型](/images/posts/nvidia_driver/cudnn_select.png)

> 注意：这里下载需要注册并登录才可以

## 安装

``` shell
sudo dpkg -i libcudnn7_7.3.0.29-1+cuda9.0_amd64.deb
```

# 最后
在 /etc/profile 内添加如下，然后 source /etc/profile，TensorFlow 中会用到

``` shell
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda/lib64"
```
