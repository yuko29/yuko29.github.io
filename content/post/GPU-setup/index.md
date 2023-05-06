---
title: "GPU Setup on Elementary OS"
date: 2023-05-06T20:16:59+08:00
categories: ['OS']
tags: ['GPU', 'CUDA']
draft: false
---

紀錄裝 Nvidia GPU driver、CUDA、CuDNN 的過程。

<!--more-->

Enviroment:
- Elementary OS 7
- Nvidia RTX 3060
- Kernel release: 5.15.0-58-generic

## Nvidia GPU driver

一開始蠻疑惑要裝哪個版本的，網上各種裝法都有，看到蠻多人說裝 recommand 的就好，於是我也照做，裝了推薦的 nvidia-driver-530-open 版。
```
$ ubuntu-drivers devices
vendor   : NVIDIA Corporation 
driver   : nvidia-driver-515 - distro non-free
driver   : nvidia-driver-530-open - distro non-free recommended
driver   : nvidia-driver-470-server - distro non-free
driver   : nvidia-driver-525-open - distro non-free
driver   : nvidia-driver-525 - distro non-free
driver   : nvidia-driver-515-open - distro non-free
driver   : nvidia-driver-530 - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : nvidia-driver-510 - distro non-free
driver   : nvidia-driver-515-server - distro non-free
driver   : nvidia-driver-525-server - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```
結果不但抓不到顯卡，連叫 nvidia-smi 還會讓 kernel 不明原因當掉...。  
說明 recommand 就只是 recommand。

最後照在 reddit 上的這篇 post: [You want to use elementary os but have a nvidia card and problems with drivers?
](https://www.reddit.com/r/elementaryos/comments/10q99ib/you_want_to_use_elementary_os_but_have_a_nvidia/) 裝了 nvidia-driver-525 版。

```
$ sudo su
$ sudo apt install linux-headers-$(uname -r)
$ apt-get install nvidia-driver-525
reboot
```
實測 nvidia-driver-525 有成功安裝並抓到顯卡。
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.105.17   Driver Version: 525.105.17   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0 Off |                  N/A |
|  0%   40C    P8    14W / 170W |      6MiB / 12288MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A      1239      G   /usr/lib/xorg/Xorg                  4MiB |
+-----------------------------------------------------------------------------+

```

## CUDA

CUDA 安裝很簡單，去[官網](https://developer.nvidia.com/cuda-toolkit-archive) 按版本抓 Runfile 下來安裝就可以了。

```
sudo sh cuda_12.0.1_525.85.12_linux.run 
```

設定環境變數。

```
export PATH="/usr/local/cuda-12.0/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-12.0/lib64:$LD_LIBRARY_PATH"
```

看有沒有抓到 nvcc。

```
$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Fri_Jan__6_16:45:21_PST_2023
Cuda compilation tools, release 12.0, V12.0.140
Build cuda_12.0.r12.0/compiler.32267302_0
```

## CuDNN

CuDNN 我是去[官網](https://developer.nvidia.com/rdp/cudnn-download) 直接下載 tar file。
解壓之後按照 [NVIDIA cuDNN Documentation](https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html#installlinux-tar) 指示把 source 和 lib 裝到 cuda 目錄下面。 

```
$ sudo cp cudnn-*-archive/include/cudnn*.h /usr/local/cuda/include 
$ sudo cp -P cudnn-*-archive/lib/libcudnn* /usr/local/cuda/lib64 
$ sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*
```