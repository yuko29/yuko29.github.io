---
title: "Git： SSH 連接至 GitHub"
date: 2020-12-10T18:54:20+08:00
slug: ""
description: ""
keywords: []
categories: ["git"]
tags: ["git"]
---

<!--more-->

### 檢查本地有沒有SSH key

```bash
$ ls -al ~/.ssh
```

如果沒有就新建一個

```bash
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

一路按 enter，將生成一個`.ssh`目錄，內有三個檔案：

* `id_rsa`(私鑰)
* `id_rsa.pub`(公鑰)
* `known_hosts`

### 在 GitHub > Settings 新增 SSH key，將公鑰 `id_rsa.pub` 貼上

#### 開啟ssh-agent

```bash
$ ssh-agent -s
```

#### 添加本地私鑰至ssh-agent

```bash
$ ssh-add ~/.ssh/id_rsa
```
#### 報錯解決
在 windows 常常會報錯，如果出現下面訊息:

```
Could not open a connection to your authentication agent
```
解決：打開 Git Bash，依序執行

```bash
$ exec ssh-agent bash
$ eval ssh-agent -s
$ ssh-add ~/.ssh/id_rsa
```
測試鍵結是否成功

```bash
$ ssh -T git@github.com
```
結束 ssh-agent（退出 shell ）

```bash
$ exit
```
