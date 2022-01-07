---
title: git_bug
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-01-07 11:22:04
---

# Git常见bug记录

<!-- more -->

#### 1. 密码权限问题

---

> 在使用https协议做push时，如果曾经使用过码云，但密码有过改动，此时会报错
>
> 之前的账号凭证与新的git账号凭证对不上，所以会出现以下问题：

|                      使用https协议报错                       |
| :----------------------------------------------------------: |
| ![Iey1TN](http://xwjpics.gumptlu.work/qinniu_uPic/Iey1TN-20220107112251542.png) |

> 解决方案:  [控制面板  》 凭据管理器 》]() 删除对应凭证，再次使用时会提示重新输入密码。
>
> 删除之前的码云凭证，然后重新push即可

#### 2. Updates were rejected because the tip of your current branch is behind

! [rejected]        master -> master (non-fast-forward)
error: failed to push some refs to 'git@github.com:xxxxxxx.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.

解决:

有如下几种解决方法：

1.使用强制push的方法：

`$ git push -u origin master -f `

这样会使远程修改丢失，一般是不可取的，尤其是多人协作开发的时候。

2.push前先将远程repository修改pull下来

`$ git pull origin master`切换

`$ git push -u origin master`

3.若不想merge远程和本地修改，可以先创建新的分支：

`$ git branch [name]`

然后push

`$ git push -u origin [name]`

#### 3. error: pathspec 'master' did not match any file(s) known to git

```shell
git branch   # 查看自己的分支,检查master是否在自己分支
git fetch		 # 构建所有的分支
git checkout mast # 转换分支
```

#### 4.remote设置了远程仓库，但是看不到分支

`git remote update origin --prune`

#### 5. fatal: Not a valid object name: ‘master‘

先add、commit再操作




