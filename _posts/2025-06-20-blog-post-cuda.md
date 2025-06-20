---
title: 'CUDA / CUDA Toolkit / CudNN /  NCCL 配置'
date: 2025-06-20
permalink: /posts/2025/06/20/cuda/
tags:
  - Notes
---

> 对于一个深耕机器学习的孩纸来说，cuda简直是噩梦的代名词，一般我们都选择继承前辈遗产或者抄菜谱，最近在实习，迫于现实无奈狠狠研究了一下CUDA各个名词之间的关系，并在此进行汇总，希望能够对读者有所帮助

TLDR

* 安装最新的CUDA Driver
* 根据约束关系安装最新的CUDA Toolkit
* 根据CUDA Toolkit，安装你需要的各种包

## 万物源头：显卡

首先我们得有一张NVIDIA显卡，显卡不依赖于任何包，只依赖于💰(bushi)。其算力情况和架构均可以在 https://developer.nvidia.com/cuda-gpus 该网站上查询。

## 显卡操作指南：CUDA Driver 

（依赖于[显卡](#万物源头：显卡)）

[【下载点击这里】](https://www.nvidia.com/en-us/drivers/unix/)

**CUDA Driver** 是 NVIDIA 提供的一部分软件，用于在计算机上运行使用 **CUDA**（Compute Unified Device Architecture）编写的程序。它是连接 **GPU 硬件** 和 **CUDA 应用程序** 的中间层，主要负责在 GPU 上执行 CUDA 程序。当你运行使用 CUDA 编写的程序时，CUDA Driver 提供运行时支持（Runtime API 是基于它工作的）。

每张显卡都有一个 **最低驱动版本要求**，通常是为了启用对应的 CUDA Compute Capability 

NVIDIA 的最新版驱动往往支持 **当前所有显卡**，甚至包括旧型号――这是它们所谓的「向下兼容性」。

所以，简单来说：CUDA Driver版本越高越好！

我们可以通过`nvidia-smi`查看CUDA Driver的版本

## 开发者工具：CUDA Toolkit

（依赖于GCC版本，[CUDA Driver](#显卡操作指南：CUDA Driver )以及[显卡](#万物源头：显卡)） 

[【下载点击这里】](https://developer.nvidia.com/cuda-downloads)

我们常常说的CUDA版本一般代指的就是此处CUDA Toolkit的版本。

CUDA Toolkit是NVIDIA 提供的开发工具包，用于在 GPU 上进行并行计算，包括nvcc 编译器，用来编译 CUDA C/C++ 代码。既然提到了C，这里很显然要依赖于GCC版本，GCC和GLIBC以及LINUX内核是绑定的，所以一般**系统决定了我们的CUDA Toolkit的版本上限**。在 [【点击这里】](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#host-compiler-support-policy) 中可以查看到最新版本的CUDA所需要的编译器版本。

同时**CUDA Toolkit必须要比CUDA Driver的版本要低**，具体关系在[【点击这里】](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#cuda-driver)有介绍。

最后CUDA Toolkit和显卡是有约束关系的，在wiki中有一张 [【矩阵图】](https://en.wikipedia.org/wiki/CUDA#GPUs_supported) 展示了这个情况。但是这个约束关系最弱，一般来说只要不是古董级显卡，CUDA Toolkit版本越高能兼容更多显卡。

我们可以通过`nvcc --version`查看CUDA Toolkit的版本。

下面给出我们自己安装的 CUDA Toolkit 和torch包中包含的 CUDA Toolkit 的区别。我们在pytorch官网中看到的CUDA版本指的就是编译torch的CUDA Toolkit版本，并且这个CUDA Toolkit会以丐版的形式安装在我们的conda环境中。但无论是哪种，我们都需要记得指定`LD_LIBRARAY_PATH=/usr/local/cuda/lib64`（这里展示的是自己安装的cuda的，conda的话是在环境下的lib64文件夹）以确保可以让程序找到动态链接库。

| **CUDA Toolkit（NVIDIA 官方）** | **conda cudatoolkit / pytorch-cuda** |
| ------------------------------- | ------------------------------------ |
| 编译/开发任何 CUDA 程序         | 运行 PyTorch 本身                    |
| nvcc、profiler、头文件、全部库  | 仅 cudart + cuBLAS/cuDNN/NCCL…       |
| ~2 GB                           | 数百 MB                              |
| 系统级目录                      | Conda/Pip 虚拟环境                   |
| 需匹配显卡驱动；占用系统 PATH   | 只需驱动；环境隔离                   |
| 编译自定义 Kernel、C++ GPU 应用 | 常规 PyTorch 训练/推理               |

## 其他内容cuDNN、NCCL等

（依赖于[CUDA Toolkit](开发者工具：CUDA Toolkit)）

### cuDNN（CUDA Deep Neural Network library）

 [【下载点击这里】](https://developer.nvidia.com/cudnn-downloads)（需登陆账号）

提供各种深度学习常用操作的高效GPU实现，例如卷积（Convolution）、池化（Pooling）、激活函数、RNN/LSTM、BatchNorm 等，加速前向/反向传播。

与CUDA toolkit的关系在 [【点击这里】](https://docs.nvidia.com/deeplearning/cudnn/backend/latest/reference/support-matrix.html)

------

### **NCCL**（NVIDIA Collective Communications Library）

 [【下载点击这里】](https://developer.nvidia.com/nccl/nccl-download)（需登陆账号）

提供多GPU/多节点之间的高性能通信原语，包含 AllReduce、Broadcast、AllGather、ReduceScatter、点对点 send/recv 等，能够自动根据 PCIe、NVLink、InfiniBand 等硬件拓扑优化通信路径与性能。

与CUDA toolkit的关系在 [【点击这里】](https://docs.nvidia.com/deeplearning/nccl/install-guide/index.html#softreq)