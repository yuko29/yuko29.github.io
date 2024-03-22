---
title: "Build TVM From Source and Codegen"
date: 2022-11-08T22:05:58+08:00
categories: ["Deep Learning"]
tags: ["build", "tvm"]
draft: false
---

最近做研究需要要看 TVM 的 CMSIS-NN backend 產生的 code，發現網路上這部份的討論不是很多。  
紀錄一下 build TVM 以及 codegen 的過程。

<!--more-->

## Build TVM

考慮到可能之後會改 TVM，所以選擇 build from source，主要按照[官方文件](https://tvm.apache.org/docs/install/from_source.html)的流程走。  
由於我習慣把各個不同的軟體建置環境隔開，避免產生依賴套件的版本衝突，所以我選擇在 conda environment 裡 build。  

1, 下載 TVM repo

```bash
git clone --recursive https://github.com/apache/tvm tvm
```

2. 建立 conda environment

```bash
# Create a conda environment with the dependencies specified by the yaml
conda env create --file conda/build-environment.yaml
# Activate the created environment
conda activate tvm-build
```

3. 編輯 cmake configuration

```bash
mkdir build
cp cmake/config.cmake build
```

這部份需要特別留意，為了讓編譯出來的 TVM 能夠支援 CMSIS-NN，需要在 `config.cmake` 修改以下相關設定。

```text
set(USE_CMSISNN ON)
set(USE_MICRO ON)
set(USE_LLVM ON)
```

其他設定可以依據需求更動。

4. Build

```bash
cd build
cmake .. -G Ninja
ninja
```

完成後會在 `build` 目錄下看到 `libtvm_runtime.so` 和 `libtvm.so` 這兩個檔案。

5. Install TVM package

官方有提供兩種作法：

- Method 1

如果會常常改動 TVM code 或設定需要重新 build 的話，可以設定 `PYTHONPATH` 直接告訴 python library 的位置。
```bash
export TVM_HOME=/path/to/tvm
export PYTHONPATH=$TVM_HOME/python:${PYTHONPATH}
```
- Method 2

建立 python binding，這個作法 TVM 會被裝在當前的環境下，所以如果是在虛擬環境下裝，離開虛擬環境後就不能使用了。
```bash
cd python
python setup.py install
```

到此 TVM 就安裝完成了。


## Compile

裝好 TVM 後就能使用 `tvmc` 來 compile model 了。  
設定 target 是 `cmsis-nn,c`，TVM 會盡可能去把 op map 到 `CMSIS-NN` library 實作 ，不能的才會用 c code。  
這裡指定 target cpu 是 `cortex-m7`。

```bash
tvmc compile --target=cmsis-nn,c \ 
    --target-cmsis-nn-mcpu=cortex-m7 \
    --target-c-mcpu=cortex-m7 \
    --runtime=crt \
    --executor=aot \
    --executor-aot-interface-api=c \
    --executor-aot-unpacked-api=1 \
    --pass-config tir.usmp.enable=1 \
    --pass-config tir.usmp.algorithm=hill_climb \
    --pass-config tir.disable_storage_rewrite=1 \
    --pass-config tir.disable_vectorize=1 \
    ./pretrainedResnet_quant.tflite \               # model
    --output-format=mlf
```

Compile 完會產生一個 TAR package `module.tar`，把它解開後會得到該 model 透過 TVM 產生出來的 library，其格式為 [Model Library Format](https://tvm.apache.org/docs/arch/model_library_format.html)，詳細的 layout 介紹可以參考文件。

```
mlperf-ic
├── codegen
│   └── host
│       ├── include
│       │   └── tvmgen_default.h
│       └── src
│           ├── default_lib0.c
│           ├── default_lib1.c
│           └── default_lib2.c
├── metadata.json
├── module.tar
├── parameters
│   └── default.params
├── runtime
└── src
    └── default.relay
```

### Codegen content

要看 model codegen 的內容，主要看 `codegen` 目錄。

```
codegen
└── host
    ├── include
    │   └── tvmgen_default.h
    └── src
        ├── default_lib0.c
        ├── default_lib1.c
        └── default_lib2.c
```

`tvmgen_default.h` 裡定義了 model 的 input/output tensor pointer 和宣告 model 的 entry point。

```c
/*!
 * \brief Input tensor pointers for TVM module "default" 
 */
struct tvmgen_default_inputs {
  void* input_1_int8;
};

/*!
 * \brief Output tensor pointers for TVM module "default" 
 */
struct tvmgen_default_outputs {
  void* Identity_int8;
};

/*!
 * \brief entrypoint function for TVM module "default"
 * \param inputs Input tensors for the module 
 * \param outputs Output tensors for the module 
 */
int32_t tvmgen_default_run(
  struct tvmgen_default_inputs* inputs,
  struct tvmgen_default_outputs* outputs
);
/*!
```

`default_lib0.c` 裡放的是 model 的權重等 constant，以及定義 entry point function。

```c
#include "tvm/runtime/c_runtime_api.h"
#ifdef __cplusplus
extern "C" {
#endif
__attribute__((section(".rodata.tvm"), ))
static struct global_const_workspace {
  int8_t constant_48_let[36864] __attribute__((aligned(16))); // 36864 bytes, aligned offset: 0
  int8_t constant_42_let[18432] __attribute__((packed, aligned(16))); // 18432 bytes, aligned offset: 36864
  int8_t constant_30_let[9216] __attribute__((packed, aligned(16))); // 9216 bytes, aligned offset: 55296
  ...
  } global_const_workspace = {
  .constant_48_let = {
    -0x3d, -0x0f, -0x12, -0x0a, -0x3e, +0x4b, +0x2b, +0x17, 
    -0x01, +0x27, -0x47, +0x1c, +0x0f, -0x17, +0x2f, -0x28, 
    -0x06, +0x17, -0x10, +0x1f, +0x00, -0x2d, +0x4c, +0x10, 
    -0x29, -0x22, -0x13, -0x01, -0x3e, -0x3e, -0x38, -0x46, 
    ...
};// of total size 84120 bytes
__attribute__((section(".bss.noinit.tvm"), aligned(16)))
static uint8_t global_workspace[49728];
#include <tvmgen_default.h>
TVM_DLL int32_t tvmgen_default___tvm_main__(void* input_1_int8,void* output0,uint8_t* global_const_workspace_0_var,uint8_t* global_workspace_1_var);
int32_t tvmgen_default_run(struct tvmgen_default_inputs* inputs,struct tvmgen_default_outputs* outputs) {return tvmgen_default___tvm_main__(inputs->input_1_int8,outputs->Identity_int8,&global_const_workspace,&global_workspace);
}
```

`default_lib1.c` 裡定義了 `tvmgen_default___tvm_main__` function，是 model inference 的主要流程。

```ｃ
TVM_DLL int32_t tvmgen_default___tvm_main__(int8_t* input_1_int8_buffer_var, int8_t* Identity_int8_buffer_var, uint8_t* global_const_workspace_0_var, uint8_t* global_workspace_1_var) {
  void* constant_28_let = (&(global_const_workspace_0_var[81968]));
  void* constant_30_let = (&(global_const_workspace_0_var[55296]));
  ...
  if (tvmgen_default_cmsis_nn_main_0(input_1_int8_buffer_var, constant_0_let, constant_1_let, constant_3_let, constant_5_let, sid_7_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_2(sid_7_let, constant_6_let, constant_7_let, constant_9_let, constant_11_let, sid_14_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_3(sid_14_let, constant_12_let, constant_13_let, constant_15_let, constant_17_let, sid_21_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_1(sid_7_let, sid_21_let, sid_22_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_5(sid_22_let, constant_18_let, constant_19_let, constant_21_let, constant_23_let, sid_29_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_7(sid_22_let, constant_24_let, constant_25_let, constant_27_let, constant_29_let, sid_36_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_8(sid_36_let, constant_30_let, constant_31_let, constant_33_let, constant_35_let, sid_43_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_6(sid_29_let, sid_43_let, sid_44_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_10(sid_44_let, constant_36_let, constant_37_let, constant_39_let, constant_41_let, sid_51_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_12(sid_44_let, constant_42_let, constant_43_let, constant_45_let, constant_47_let, sid_58_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_13(sid_58_let, constant_48_let, constant_49_let, constant_51_let, constant_53_let, sid_65_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_11(sid_51_let, sid_65_let, sid_66_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_15(sid_66_let, sid_67_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_16(sid_67_let, constant_54_let, constant_55_let, sid_70_let, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  if (tvmgen_default_cmsis_nn_main_17(sid_70_let, Identity_int8_buffer_var, global_const_workspace_0_var, global_workspace_1_var) != 0 ) return -1;
  return 0;
}
```

`default_lib2.c`　裡定義了每個 operator 的實作會 bind 到哪個 CNSIS-NN 的 operator 實作上。

```c
...
TVM_DLL int32_t tvmgen_default_cmsis_nn_main_0(int8_t* input_, int8_t* filter_, int32_t* multiplier_, int32_t* bias_, int32_t* shift_, int8_t* output_, uint8_t* global_const_workspace_2_var, uint8_t* global_workspace_3_var) {
  void* context_buffer_0_let = (&(global_workspace_3_var[16384]));
  cmsis_nn_context context= {context_buffer_0_let,108};
  cmsis_nn_tile stride = {1,1};
  cmsis_nn_tile padding = {1,1};
  cmsis_nn_tile dilation = {1,1};
  cmsis_nn_activation activation = {-128,127};
  cmsis_nn_conv_params conv_params = {128, -128, stride, padding, dilation, activation};
  cmsis_nn_per_channel_quant_params quant_params = {multiplier_, shift_};
  cmsis_nn_dims input_dims = {1,32,32,3};
  cmsis_nn_dims filter_dims = {16,3,3,3};
  cmsis_nn_dims bias_dims = {1,1,1,16};
  cmsis_nn_dims output_dims = {1,32,32,16};
  arm_cmsis_nn_status status = arm_convolve_wrapper_s8(&context, &conv_params, &quant_params, &input_dims, input_, &filter_dims, filter_, &bias_dims, bias_, &output_dims, output_);
  if (status != ARM_CMSIS_NN_SUCCESS) {
    return -1;
  }
  return 0;
}
...
```

## Reference

[TVM - Install from Source](https://tvm.apache.org/docs/install/from_source.html)  
[TVM - Running TVM on bare metal Arm(R) Cortex(R)-M55 CPU and Ethos(TM)-U55 NPU with CMSIS-NN](https://tvm.apache.org/docs/how_to/work_with_microtvm/micro_ethosu.html)  
[TVM - Model Library Format](https://tvm.apache.org/docs/arch/model_library_format.html)  
[Running TVM on bare metal Arm(R) Cortex(R)-M55 CPU and CMSIS-NN](https://github.com/apache/tvm/tree/main/apps/microtvm/cmsisnn)  
