---
title: python机器学习常用命令
tags:
  - python
categories:
  - technical
  - null
toc: true
declare: true
date: 2021-08-08 14:32:41
---

# Conda

```shell
# 创建制定python版本的环境
conda create --name your_env_name python=3.7
# 获取当下所有环境
conda info --envs
conda env list
# 进入某一个环境
activate your_env_name
# 退出环境
deactivate
# 复制某个环境
conda create --name new_env_name --clone old_env_name 
# 删除某个环境
conda remove --name your_env_name --all
```

<!-- more -->

