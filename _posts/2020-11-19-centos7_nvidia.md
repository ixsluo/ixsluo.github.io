---
layout: post
title: Centos7 Nvidia Driver
subtitle: 
author: ixsluo
date: 2020-11-19 15:37:56 +0800
categories: 
tag: 
---

Centos7离线安装Nvidia显卡驱动

- [环境需求](#环境需求)
- [Nvidia驱动(450)](#nvidia驱动450)
- [CUDA(11.0)](#cuda110)
- [cuDNN](#cudnn)

## 环境需求

1. 依赖包
```bash
yum install gcc kernel-devel kernel-headers
```

2. 检查内核版本以及/boot目录大于300M
```bash
uname -r  # 3.10.*
df -h
```

3. 查看显卡(Tesla K80)
```bash
lspci | grep NVIDIA
```

4. 禁用nouveau

- 查看nouveau是否启用
```bash
lsmod | grep nouveau
# 无输出说明已禁用
```
- 禁用nouveau
```bash
echo blacklist nouveau >> /etc/modprobe.d/blacklist-nouveau.conf
echo option nouveau modeset=0 >> /etc/modprobe.d/blacklist-nouveau.conf
dracut --force  # 重新生成kernel initramfs
reboot
```

## Nvidia驱动(450)

- [Nvidia驱动下载](https://www.nvidia.cn/Download/index.aspx?lang=cn)

1. 驱动安装
```bash
chmod +x ~/apps/NVIDIA-Linux-*.run
./apps/NVIDIA-Linux-*.run --kernel-source-path=/usr/src/kernels/3.10.*** -k $(uname -r) --no-drm
nvidia-smi  # 查看是否安装成功
```

2. [部分错误及解决方法参考](https://www.cnblogs.com/2012blog/p/9431432.html)

- X library字符模式警告可忽略


## CUDA(11.0) 

- [Nvidia下载地址](https://developer.nvidia.com/cuda-downloads)
- [官方文档](https://docs.nvidia.com/cuda/)

1. 安装
```bash
chmod +x ~/apps/cuda_11.*.run
./apps/cuda_11.*.run --kernel-source-path=/usr/src/kernels/3.10.*** --no-drm
```
不必再次安装Driver，Sample、Demo、Doc可选则安装

选择完毕后`install`

2. PATH及链接
```bash
echo export PATH=/usr/local/cuda-11.0/bin:$PATH >> /etc/profile
source /etc/profile
echo /usr/local/cuda-11.0/lib64 >> /etc/ld.so.conf
ldconfig
```

3. 卸载
```bash
./usr/local/cuda-11.0/bin/cuda-uninstaller
```

## cuDNN

- [官方文档](https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html)

```bash
tar -xzvf cudnn-x.x-linux-x64-v8.x.x.x.tgz
cp cuda/include/cudnn*.h /usr/local/cuda/include
cp cuda/lib64/libcudnn* /usr/local/cuda/lib64
chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
```