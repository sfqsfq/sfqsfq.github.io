---
title: "Github Action"
date: 2023-03-01T17:36:29+08:00
draft: false
---

# 工作流创建

1. 创建 `.github/workflows` 目录
2. 在 `.github/workflows` 目录下创建 `main.yml` 文件 
3. 在 `main.yml` 文件中添加如下内容

```yml
name: Github Actions Demo
run-name: ${{ github.actor }} is testing out Github Actions
on: [push]

jobs:
  Explore-Github-Actions:
    runs-on: ubuntu-latest
    steps:
      - run: echo "🎉 The job was automatically triggered by a ${{ github.event_name }} event."
      - run: echo "🐧 This job is now running on a ${{ runner.os }} server hosted by GitHub!"
      - run: echo "🔎 The name of your branch is ${{ github.ref }} and your repository is ${{ github.repository }}."
      - name: Check out repository code
        uses: actions/checkout@v3
      - run: echo "💡 The ${{ github.repository }} repository has been cloned to the runner."
      - run: echo "🖥️ The workflow is now ready to test your code on the runner."
      - name: List files in the repository
        run: |
          ls ${{ github.workspace }}
      - run: echo "🍏 This job's status is ${{ job.status }}."
```

4. 提交代码到 `github` 仓库