---
title: "Building Tensorflow From Source"
date: 2021-09-12T18:21:01+08:00
categories: ["Deep Learning"]
tags: ["build", "tensorflow"]
draft: false

---

紀錄一下自己 build tensorflow 的經過。

<!--more-->

安裝過程主要參考 kmhofmann 寫的這篇：[Building TensorFlow from source (TF 2.3.0, Ubuntu 20.04)](https://gist.github.com/kmhofmann/e368a2ebba05f807fa1a90b3bf9a1e03)

官方文件在此：[Build from source](https://www.tensorflow.org/install/source)

## Configuration

- Ubuntu 20.04
- Tensorflow v2.7.0

- gcc version 9.3.0 (Ubuntu 9.3.0-17ubuntu1~20.04)

本次只建構 CPU support (default) 的部分，需要 CUDA support 的話需要下載 NVIDIA driver、 CUDA 和 cuDNN，安裝過程可以參考同篇作者寫的這篇：[Installing the NVIDIA driver, CUDA and cuDNN on Linux (Ubuntu 20.04)](https://gist.github.com/kmhofmann/cee7c0053da8cc09d62d74a6a4c1c5e4)

## Install Bazelisk

Bazel 是 Google 開發的自動化軟體建構工具，可以在多種語言平台上建置，詳細特色可以查閱它的[官網](https://bazel.build/)。

官方文件推薦使用 Bazelisk 來下載 Bazel，Bazelisk 是 Bazel 的 launcher，能自動下載符合當下環境的 Bazel 版本。

下載 Bazelisk 的方式有很多種，可以參考 [Bazel 官網說明](https://docs.bazel.build/versions/main/install-bazelisk.html) 和 [GitHub](https://github.com/bazelbuild/bazelisk)。

我這次是直接下載 binary [
Releases · bazelbuild/bazelisk - GitHub](https://github.com/bazelbuild/bazelisk/releases)，選擇 `bazelisk-linux-amd64` 版本的。

下載後把它移到 `/usr/local/bin`，改名成 `bazel`，並確定路徑有被加進 PATH。  

```bash
$ mv bazelisk-linux-amd64 /usr/local/bin/bazel
```

## Build Tensorflow

下載 Tensorflow source code

```bash
$ git clone https://github.com/tensorflow/tensorflow.git
$ cd tensorflow
```

開 python virtual environment，然後 activate 它

```bash
$ python3 -m venv ~/.virtualenvs/tf_build
$ source ~/.virtualenvs/tf_dev/bin/activate
```

下載 building 所需要的 python package，參照 [Tensorflow documentation](https://www.tensorflow.org/install/source)

```bash
$ pip install -U pip numpy wheel
$ pip install -U keras_preprocessing --no-deps
```

設定 Bazel build configuation options

```bash
$ python configure.py
```

這裡會問許多問題，確定 python package 位置在 virtual environment 後，之後的選項都可以略過 (N)

完成之後就可以開始 build 了：

```bash
$ bazel build --config=opt -c opt //tensorflow/tools/pip_package:build_pip_package
```

過程要花很多時間，如果都沒有發生 ERROR，看到 Build completed successfully 就表示成功了！

```
INFO: Build completed successfully, 11496 total actions
```

## Build and Install Tensorflow Python Package

前一步產生的是 build_pip_package 執行檔，執行它在目標目錄可以產生 `.whl`

```bash
$ ./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

上面指令完成後會在`/tmp/tensorflow_pkg` 產生 tensorflow 的 `.whl`，根據 tensorflow 版本、python 版本和系統不同檔名會有所不同，我的話是 `tensorflow-2.7.0-cp38-cp38-linux_x86_64.whl` 

下載這個 tensorflow wheel

```bash
$ pip install /tmp/tensorflow_pkg/tensorflow-2.7.0-cp38-cp38-linux_x86_64.whl
```

到這裡就完成 tensorflow 安裝了

## Test

可以用以下 command 測試 tensorflow 是否正常運作

```bash
$ python -c "import tensorflow as tf;print(tf.reduce_sum(tf.random.normal([1000, 1000])))"
```

它會列出所有 initialization message，可藉此知道 libarary 是否能被順利 load 進來，並跑出 sum。

如果有跑出 sum 就表示成功了 :D

```
2021-09-12 02:49:21.028307: I tensorflow/core/platform/cpu_feature_guard.cc:151] This TensorFlow binary is optimized with oneAPI Deep Neural Network Library (oneDNN) to use the following CPU instructions in performance-critical operations:  SSE3 SSE4.1 SSE4.2 AVX AVX2 FMA
To enable them in other operations, rebuild TensorFlow with the appropriate compiler flags.
tf.Tensor(1755.6423, shape=(), dtype=float32)
```

## Error

我在 Test 的過程中發生了下面錯誤：

```
RuntimeError: module compiled against API version 0xe but this version of numpy is 0xd
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/home/yuko29/.virtualenvs/tf_dev/lib/python3.8/site-packages/tensorflow/__init__.py", line 41, in <module>
    from tensorflow.python.tools import module_util as _module_util
  File "/home/yuko29/.virtualenvs/tf_dev/lib/python3.8/site-packages/tensorflow/python/__init__.py", line 41, in <module>
    from tensorflow.python.eager import context
  File "/home/yuko29/.virtualenvs/tf_dev/lib/python3.8/site-packages/tensorflow/python/eager/context.py", line 38, in <module>
    from tensorflow.python.client import pywrap_tf_session
  File "/home/yuko29/.virtualenvs/tf_dev/lib/python3.8/site-packages/tensorflow/python/client/pywrap_tf_session.py", line 23, in <module>
    from tensorflow.python.client._pywrap_tf_session import *
ImportError: SystemError: <built-in method __contains__ of dict object at 0x7f0b8ecb9040> returned a result with an error set
```

查了一下，找到了這個 [issue](https://github.com/freqtrade/freqtrade/issues/4281)，原因可能是 numpy 版本需要更新。我原本的 numpy 版本是 1.19.5，執行 `pip install numpy --upgrade` 更新到 1.21.2 就好了。
