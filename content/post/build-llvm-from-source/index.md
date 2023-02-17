---
title: "Build LLVM and clang From Source"
date: 2023-02-16T23:19:09+08:00
categories: ["LLVM"]
tags: ["build", "clang", "LLVM"]
draft: false
---

紀錄 build LLVM 和 Clang 的過程。

<!--more-->

過程主要參考這篇：[編譯 LLVM 與 Clang](https://tclin914.github.io/a0c0fe90/)

## Prepare

工具部份
- cmake
- ninja

從 github 下載 LLVM source code
```
git clone https://github.com/llvm/llvm-project.git
```
Build LLVM 和 clang 需要足夠的 disk 空間，先紀錄最後 build 完，install 完後各自佔據的空間大小。
- Build: 81G
- Install：56G

## Build & install

1. Configure
    - `-DCMAKE_INSTALL_PREFIX`：設定 LLVM install 的目錄位置
    - `-DLLVM_ENABLE_PROJECTS`：如果要編譯 clang 需要在這裡指定，如果沒有指定這個 config 的話只會 build LLVM 而已
    - `-DCMAKE_BUILD_TYPE`：必須指定，因為之後需要修改所以我選 Debug
    - `-DLLVM_TARGETS_TO_BUILD=X86`：指定要編譯那一個指令集架構 target，沒指定的話會 build 所有的 target

更多的 config 可以查詢 [Building LLVM with CMake](https://llvm.org/docs/CMake.html) 或 [Getting Started with the LLVM System](https://llvm.org/docs/GettingStarted.html)

```
$ cmake -DCMAKE_INSTALL_PREFIX="../llvm-install" -DLLVM_PARALLEL_LINK_JOBS=2 \
-DLLVM_ENABLE_PROJECTS=clang -DLLVM_TARGETS_TO_BUILD=X86 -G Ninja 
-DLLVM_ENABLE_ASSERTIONS=Yes -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS="-O0 -g3" \
-DCMAKE_CXX_FLAGS="-O0 -g3" ../llvm
```

2. Build
根據 CPU core 數決定要用幾個 core 平行跑。
```
$ ninja -j8
```
同時需要注意 RAM 夠不夠。Ninja 預設是用全部的 core 下去跑的，RAM 不夠的話會卡死失敗。
```
collect2: fatal error: ld terminated with signal 9 [Killed]
```
如果出現這個 error 就是 RAM 不夠用了，可以選擇用少一點的 core 跑或增加除夠的 swap 解決。 

3. Install
把 LLVM/Clang 安裝到指定的目錄裡。
```
$ ninja install
```

如果沒指定 install 位置的話預設是裝在 `/usr/local/`，所以會裝在 root 目錄下，如果一開始 root 的分區不夠大的話可能就會裝不下。要注意 LLVM 這邊沒有提供 uninstall 功能可以用，所以要清理裝到一半的檔案的話會非常痛苦，要擴充 root 的話又是一個故事了。 