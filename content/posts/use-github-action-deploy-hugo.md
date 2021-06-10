---
title: "Use GitHub Action 將 Hugo 部署到 GitHub Page"
date: 2021-06-10T02:30:43+08:00
categories: ['DevOps']
tags: ['github', 'hugo']
draft: false
---

想要接觸 CI/CD，GitHub 有提供 CI/CD 服務 GitHub Actions，於是先拿自己的 blog 下手。  
紀錄如何透過 GitHub Action 將 Hugo 部署到 GitHub Page 上。

<!--more-->

一般來說，要將 Hugo 部署到 GitHub Page 需要兩個步驟：

- 用 `hugo` 命令產生 `./public` 資料夾，裡面將會是渲染過後的靜態網站內容
- 將 `./public` 資料夾的內容傳到 GitHub 上

當然這些也都可以透過寫 script 來自動化部署過程，但這次想趁這個機會學習 GitHub Action 的用法。由於我想要在同一個 repo 維護 blog 的 source files，於是我決定要設定一個 GitHub Action，在我將原本 Hugo 的 source files 推上到 main 分支時，自動幫我完成產生上述的兩個步驟，將靜態網站內容寫到 gh-page 分支。

## GitHub Actions 的基本概念

GitHub Actions 所需具備的元素，階層由上到下依序是：Workflow、Job、Step、Action：

- Workflow：規劃 CI/CD 的工作流程，由一個或多個 Job 組成
- Job：任務，由多個 Step 組成
- Step：步驟，由多個 Action 組成
- Action：動作，Step 可以執行多個 Action
  
若要使用 GitHub Actions，只需要在專案中的 `.github/workflows` 目錄撰寫 `.yml` config 檔。  
在專案被 push 到 GitHub 時，GitHub 就會執行 `.github/workflows` 內所有的 YMAL config 文件。

## 建立 Workflow

以下是拿 Hugo 官方提供的範例進行修改的。  
在專案目錄內建立 `.github/workflows/gh-page.yml`。  
為 workflow 取名：

```yml
name: github pages  # set workflow name
```

設定觸發 Action 的時機以及分支：這裡設定在要在 push 上 main 分支時觸發 Action（分支可自行決定）。

```yml
on:
  push:
    branches:
      - main    # Set a branch name to trigger deployment
  pull_request:
```

## 建立 Job

建立一個 job，為這個 job 命名，並設定這個 job 要跑在什麼環境下：  

```yml
jobs:
  deploy:   # set the name of the job
    runs-on: ubuntu-18.04   # running at ubuntu-18.04
```

設定 job 中的步驟：

- 首先要 checkout repository，讓我們撰寫的 workflow 可以存取它。
- 這裡使用 GitHub 官方的 [actions/checkout@v2](https://github.com/actions/checkout)
- 如果 blog 的 theme 是用 submodule 載入的，記得要將 `submodules` 設為 `true`，不然會因為推上 repo 的檔案中沒有 submodule 的檔案而找不到 theme。

```yml
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
```

- Setup Hugo
  - 這裡使用 [peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo#getting-started)
  - 指定 Hugo 版本（`hugo-version`)：可以指定特定版本或是直接用最新版 `latest`
  - 以及是否要使用 Hugo extended（`extended`）：如果有使用 sass/scss 要設定為 true
- Build
  - 建置 hugo 網站，也就是原本在本機上產生 `./public` 的動作，這裡就依照自己的需求寫。

```yml
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: latest      # specify the version of Hugo
          extended: true            # whether need the Hugo extended

      - name: Build
        run: hugo --minify --gc     # build the website
```

- Deploy
  - 將 Hugo 建置出來的網站部署到 GitHub Page
  - 使用[peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) 完成
  - `github_token`：設定推送用 token，由 action 處理，必填
  - `publish_branch`：設定要部署到哪個目標分支
  - `force_orphen`：目標分支是否要是獨立分支，只保留最後一個 commit

```yml
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages  # default: gh-pages
          force_orphan: true 

```

整個 workflow：

```yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: latest
          extended: true

      - name: Build
        run: hugo --minify --gc

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
            github_token: ${{ secrets.GITHUB_TOKEN }}
            publish_branch: gh-pages  # default: gh-pages
            force_orphan: true

```

接著只要在 blog 資料夾中寫完文章，commit 完並 push 到 repo 的 main 分支後就可以觸發這個 Action 把網站內容建置出來並部署到 `gh-pages` 分支上了。

另外要記得把 GitHub Page 的 Source 分支選為 `ph-pages`（`repo` > `Setting > Pages > Source`），這樣 GitHub Page 才會讀到剛剛部署的網站內容。

## 參考資料

- [Host on GitHub - Hugo 官方教學](https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action)
- [使用 Github Actions 來自動化部署 Hugo 到 Github Pages - 作者：Puck Wang](https://blog.puckwang.com/post/2020/use-github-actions-deploy-hugo/)
- [peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo#getting-started)
- [實作開源小工具，與 Github Actions 的第一次相遇！ - 作者：莫力全 Kyle Mo](https://medium.com/starbugs/%E5%AF%A6%E4%BD%9C%E9%96%8B%E6%BA%90%E5%B0%8F%E5%B7%A5%E5%85%B7-%E8%88%87-github-actions-%E7%9A%84%E7%AC%AC%E4%B8%80%E6%AC%A1%E7%9B%B8%E9%81%87-3dd2d70eeb)  