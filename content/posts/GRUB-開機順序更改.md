---
title: "GRUB 開機順序更改"
date: 2021-08-28T22:11:21+08:00
categories: ['OS']
tags: ['grub']
draft: false
---

我有一台筆電上裝有 Windows 和 Ubuntu 雙系統，GRUB 開機預設是 Ubuntu（在 GRUB 開機清單最上面），所以開機後放著不動的話會以 Ubuntu 開機。  
平常比較長使用 Windows，本來不怎麼在意開機順序的，但是 windows 常常自己更新，害的常常我一覺醒來要繼續工作時發現電腦上開著的是 Ubuntu，只好重新開機切換。  
同樣的事一直發生，久而久之讓我覺得很煩，這次決定要來好好解決這個問題。  

<!--more-->

查資料的過程中有找到兩種方法，其實應該是一體兩面，我使用的是第一種。

## 第一種：修改 `/etc/default/grub`

開啟 GURB 設定檔

```bash
sudo gedit /etc/default/grub
```

找到 `GRUB_DEFAULT` 這項設定（如下）。
```
GRUB_DEFAULT="0"
```
預設 GRUB 會開啟清單上第一個位置的作業系統（第一位為位置 `0`，往下遞增）。
把數字改成 `Windows Boot Manager` 在 GRUB 開機清單上的位置，以我的狀況 `Windows` 是在第三個位置，所以要改成 `2`。

修改後儲存，並使 GRUB 新設定生效：
```
sudo update-grub
```
## 第二種：修改 `/boot/grub/grub.cfg`

在 `/boot/grub/grub.cfg` 找到下面這行：
```
set default = “0”
```

每一個開機選項都以 `menuentry` 開頭設定寫在下方。每個開頭是 `menuentry` 的設定都是代表 GRUB 列表上的一個選項。
找到 `Windows Boot Manager` 是在第幾個之後把 `set default` 的數字改掉存檔即可，這邊我的 `Windows` 同樣是在第三個位置。
```
menuentry 'Windows Boot Manager (on /dev/nvme0n1p1)' --class windows --class os $menuentry_id_option 'osprober-efi-AE67-8287' {
        insmod part_gpt
        insmod fat
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root  AE67-8287
        else
          search --no-floppy --fs-uuid --set=root AE67-8287
        fi
        chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
set timeout_style=menu
if [ "${timeout}" = 0 ]; then
  set timeout=10
fi

```

重新開機後就可以看到 GRUB 預設改成 Windows 了！

## 參考資料
- [Linux GRUB 開機順序設定](https://tomkuo139.blogspot.com/2017/07/linux-grub.html)
- [Ubuntu16.04 GRUB開機選單順序](https://blog.twshop.asia/ubuntu16-04-grub%E9%96%8B%E6%A9%9F%E9%81%B8%E5%96%AE%E9%A0%86%E5%BA%8F/)