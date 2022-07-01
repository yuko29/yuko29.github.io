---
title: "裁剪 ONNX 模型"
date: 2022-06-30T02:23:09+08:00
categories: ['Deep Learning']
tags: ['onnx']
draft: false
---

手邊的專題需要用 ResNet 模型驗證 DLA simulator 運算正確性，但是因為模型太大，希望可以把模型依照 stage 分割，方便漸進式測試。為了達成這件事查了不少資料，順手紀錄一下分割的過程。

<!--more-->

## ONNX 格式

切割之前要先了解 ONNX 是怎麼儲存模型資料的，可以參考下面幾篇文章：

- [ONNX学习笔记](https://zhuanlan.zhihu.com/p/346511883)
- [Day 28: 再造訪 ONNX Graph](https://ithelp.ithome.com.tw/articles/10227825)

關於 ONNX 詳細的格式訊息詳細的寫在這份文件：

- [onnx.proto](https://github.com/onnx/onnx/blob/main/onnx/onnx.proto)

關於 Protocol Buffer 定義方式，可以參考 Google 的文件：

- [Protocol Buffer Basics: C++](https://developers.google.com/protocol-buffers/docs/cpptutorial)

根據上面的資訊，當載入一個 ONNX 模型，它的結構大概如下

- model：ModelProto 類型，包含一些模型的版本訊息和一個 GraphProto
  - graph：GraphProto 類型，裡面包含 4 個 repeated 數組（可視為 dynamically sized array）
    - node： NodeProto 類型，裡面存放所有的 compute operator
    - input： ValueInfoProto 類型，裡面放了模型的 input operator
    - output： ValueInfoProto 類型，裡面放了模型的 output operator
    - initializer： TensorProto 類型，裡面放了所有的權重參數

## 裁剪

知道了 ONNX 的結構後，就能很容易的規劃裁剪模型的步驟了，裁剪 ONNX 模型的步驟可如下：

1. 針對 node 取出需要的 compute operator，把其他不用的移除掉
2. 製作新的 output operator 並替換上去
3. 不用的權重參數可以從 initializer 內移掉（這次我沒有這麼做，因為我不需要）

我要裁剪的 model 是 ResNet V1 的 [ResNet18](https://github.com/onnx/models/tree/main/vision/classification/resnet)，來自 ONNX Model Zoo。

Ok，接下來就可以開始切了。
以下說明 operator 和 node 會混著用，兩者是同一個意思。

首先先載入 model。

```python
import onnx
model = onnx.load('resnet18-v1-7/resnet18-v1-7.onnx')
```

接著取出需要的 compute operator，清除掉原本的 node 後再把需要的加回去。
這裡需要知道這些 operator 的 index 位置，我是沿著 Add 這個 operator 切，第一個 Add operator 是 index 10 的 node。

```python
oldnodes = [n for n in model.graph.node]
newnodes = oldnodes[0:10] # stage1_plus0
del model.graph.node[:] 
model.graph.node.extend(newnodes) 
```

接著製作新的 output node。注意 name 和 size 都必須 match 最後一個 compute operator 的 output。

```python
new_output_node = onnx.helper.make_tensor_value_info('resnetv15_stage1__plus0', TensorProto.FLOAT, [1, 64, 56, 56]) # stage1_plus0
del model.graph.output[:]
model.graph.output.extend([new_output_node])
```

關於 output node 的 size，當然可以用算的，不過我推薦用 shape inference 跑一次原本的 ONNX 模型直接拿所有的 compute operator 的 output size。

```python
inferred_model = onnx.shape_inference.infer_shapes(model)
print(inferred_model.graph.value_info)

# ...
# name: "resnetv15_stage1__plus0"
# type {
#   tensor_type {
#     elem_type: 1
#     shape {
#       dim {
#         dim_value: 1
#       }
#       dim {
#         dim_value: 64
#       }
#       dim {
#         dim_value: 56
#       }
#       dim {
#         dim_value: 56
#       }
#     }
#   }
# ...
```

最後把修改完的模型儲存起來就大功告成了。

```python
onnx.save(model, 'resnet18-v1-7/stage1_plus0.onnx')
```

## 結尾

找資料的途中找到了很多大大寫的關於 ONNX 改圖的 script，像是  
https://blog.csdn.net/xxradon/article/details/104715524  
https://github.com/saurabh-shandilya/onnx-utils

最後才發現有個工具叫 [onnx-modifier](https://github.com/ZhangGe6/onnx-modifier) 可以用 UI 介面來修改 ONNX model，可以增減 node，看起來對於小規模的 ONNX 模型修改十分方便。用這個工具來修改 ResNet18 應該能很容易達到我要的目標 XD。  
不過藉由這次機會深入了解 ONNX 的格式，對未來如果需要做更進階的操作應該會挺有幫助的。
