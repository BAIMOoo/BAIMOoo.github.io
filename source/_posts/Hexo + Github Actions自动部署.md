---
title: Hexo + Github Actions自动部署
date: 2025-03-12 16:34:11
tags:
---
跟着官方文档部署的Hexo：[在 GitHub Pages 上部署 Hexo](https://hexo.io/zh-cn/docs/github-pages)


使用教程默认工作流，通常每次在本地发布新博客，都需要手动执行以下命令：
1. ```hexo new post <title>``` 
   实际这一步就只是在source\posts文件夹下创建了一个md文件，直接手动创建，效果一样
2. ```hexo clean ```
3. ```hexo g``` 
4. ```git add .``` 
5. ```git commit -m 'xxxxx'``` 
6. ```git push origin master```

究极无敌麻烦！！！<!-- more -->碰巧在部署项目的时候，第一次了解到了Github Actions，第一印象就是一个远程服务器，能够执行大部分原本手动输入在cmd命令窗口中的指令，感觉多半是可以把第2-6步全都集成到一个工作流中，这样我就不用每次发布新博客时都手动执行这些命令了，顺带我甚至不必拉取远程仓库，只需要在仓库web页面新建或者修改source文件夹下的md文件，就能够自动发布到Github Pages了。

修改后的工作流：

```
name: Pages

on:
  push:
    branches:
      - master # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          submodules: recursive

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Cache NPM dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache

      - name: Install Dependencies
        run: npm install

      - name: Clean Hexo
        run: npx hexo clean

      - name: Generate Hexo site
        run: npx hexo generate

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
      contents: write # Add this to allow writing to the repository
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          ref: master

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install Dependencies
        run: npm install

      - name: Clean Hexo
        run: npx hexo clean

      - name: Generate Hexo site
        run: npx hexo generate

      - name: Add generated files
        run: git add .

      - name: Commit changes
        run: git commit -m "Automated commit for Hexo build"
        env:
          GIT_AUTHOR_NAME: github-actions
          GIT_AUTHOR_EMAIL: github-actions@github.com
          GIT_COMMITTER_NAME: github-actions
          GIT_COMMITTER_EMAIL: github-actions@github.com

      - name: Push changes
        if: github.event_name == 'push' && github.ref == 'refs/heads/master' && github.actor != 'github-actions[bot]'
        run: |
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git push origin master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

