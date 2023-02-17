---
title: "Xv6 Environment Setup"
date: 2022-07-27T02:33:17+08:00
categories: ['xv6']
tags: []
draft: false
---

紀錄在 Mac 系統上安裝 Xv6 環境的過程。  
主要處理 QEMU 版本過高的問題。

<!--more-->

原本是照著課程網站提供的[環境建置文件](https://pdos.csail.mit.edu/6.S081/2020/tools.html)，在 Mac 上安裝環境，但是裝完之後執行 `make qemu` 會卡在下面這行：

```bash
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive
```

主要原因是 brew 安裝的 qemu 版本過高（我裝的結果是 7.0.0），課程網站上有提供解法，驗證可成功執行的版本為 5.1.0，為了省麻煩決定用 docker 來重新安裝環境。

我主要參考這篇用 docker 安裝 xv6 環境： [xv6-riscv环境搭建](https://groverzhu.github.io/2021/08/17/xv6-riscv%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/#more)

我按照上面的方法用 apt 安裝 qemu 版本還是過高 (記得是 6.x)，仍然會卡死，所以 qemu 還是需要自己手動 build 版本為 5.1.0 的版本。

下面是我在 docker 內安裝的過程。

1. 下載 xv6 github repo

```bash
$ git clone git://g.csail.mit.edu/xv6-labs-2020
```

2. 建立 docker container，掛載 `xv6-labs-2020/` 到 container 內的目錄下  
例如：

```bash
$ docker run -itd --name=xv6-riscv -v $HOME/xv6-labs-2020:/xv6 ubuntu /bin/bash
```

3. 安裝所需要到 toolchain

```bash
$ apt install gcc-riscv64-unknown-elf git build-essential gdb-multiarch gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

4. Build qemu 5.1.0

需要先裝以下套件：
```bash
$ apt-get install libglib2.0-dev
$ apt-get install libpixman-1-dev
```
如果沒有的話可能會遇到下面幾個 error：
```bash
ERROR: glib-2.48 gthread-2.0 is required to compile QEMU
```
```bash
ERROR: pixman >= 0.21.8 not present.
       Please install the pixman devel package.
```

接著就可以按照課程網站的指示安裝 qemu 5.1.0

```bash
$ wget https://download.qemu.org/qemu-5.1.0.tar.xz
$ tar xf qemu-5.1.0.tar.xz
$ cd qemu-5.1.0
$ ./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"
$ make
$ sudo make install
$ cd ..
```
安裝位置在 /usr/local/bin，需要把這個位置加入 PATH 裡。

5. 檢查安裝沒有問題後，就可以到 xv6 的目錄下試著 boot xv6 了

```bash
$ git checkout util
$ make qemu
```

成功的話會執行結果會長這樣，發現我們成功拿到 shell。

```bash
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ 
```

可以打入指令。

```bash
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2059
xargstest.sh   2 3 93
cat            2 4 24168
echo           2 5 22984
forktest       2 6 13208
grep           2 7 27472
init           2 8 23728
kill           2 9 22936
ln             2 10 22768
ls             2 11 26360
mkdir          2 12 23072
rm             2 13 23056
sh             2 14 41888
stressfs       2 15 23928
usertests      2 16 148344
grind          2 17 38032
wc             2 18 25256
zombie         2 19 22320
console        3 20 0
```

按 `Ctrl-a x` 可以退出 xv6，至此就確定 xv6 環境安裝完成了。  