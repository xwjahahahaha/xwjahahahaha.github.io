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

# Git错误记录

<!-- more -->

## 1. 密码权限问题

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

## 2. Updates were rejected because the tip of your current branch is behind

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

## 3. error: pathspec 'master' did not match any file(s) known to git

```shell
git branch   # 查看自己的分支,检查master是否在自己分支
git fetch		 # 构建所有的分支
git checkout mast # 转换分支
```

## 4.remote设置了远程仓库，但是看不到分支

`git remote update origin --prune`

## 5. fatal: Not a valid object name: ‘master‘

先add、commit再操作

## 6.warning: LF will be replaced by CRLF in 解决办法

原因是存在符号转义问题

windows中的换行符为 CRLF， 而在linux下的换行符为LF，所以在执行add . 时出现提示，解决办法：、

`git config --global core.autocrlf false`

# 常见操作

## 1. 删除忘记忽略的文件

```shell
git rm -rf --cached .  # . 表示所有缓存，如果只要一个文件则可以选择对应文件
git add .
git commit 
git push
```

## 2. 修改过去的commit信息

https://cloud.tencent.com/developer/article/1730774

```shell
# 列出 rebase 的 commit 列表，不包含 <commit id>
$ git rebase -i <commit id>
# 最近 3 条
$ git rebase -i HEAD~3
# 本地仓库没 push 到远程仓库的 commit 信息
$ git rebase -i

# vi 下，找到需要修改的 commit 记录，```pick``` 修改为 ```edit``` 或 ```e```，```:wq``` 保存退出
# 重复执行如下命令直到完成
$ git commit --amend --message="modify message by daodaotest" --author="jiangliheng <jiang_liheng@163.com>"
$ git rebase --continue

# 中间也可跳过或退出 rebase 模式
$ git rebase --skip
$ git rebase --abort

# 最后强制push
$ git push -f
```

## 3. 修改git的默认commit编辑器为vim

`git config --global core.editor vim`

## 4. git引入子模块



## 5. 删除本地分支、远程分支

```bash
// 删除本地分支
git branch -d localBranchName

// 删除远程分支
git push origin --delete remoteBranchName
```

## 6. 回退git commit –amend

https://www.jianshu.com/p/97341ed9d89e

`git reset HEAD@{1}`

## 7.配置两台服务器ssh免密码链接

```shell
A 服务器：
#(连续三次回车,即在本地生成了公钥和私钥,不设置密码)
[root@localhost ~]# ssh-keygen -t rsa
#(需要输入密码--如果B 服务器上存在.ssh文件夹，跳过此步)
[root@localhost ~]# ssh root@B服务器I -p 22 "mkdir .ssh"
[root@localhost ~]# scp -P 22 ~/.ssh/id_rsa.pub root@B服务器IP:.ssh/id_rsa.pub

B 服务器：
# touch /root/.ssh/authorized_keys (如果已经存在这个文件, 跳过这条)
# cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys (将id_rsa.pub的内容追加到authorized_keys 中)
```

## 8. commit格式规范

**commit message格式**

```text
<type>(<scope>): <subject>
```

**type(必须)**

用于说明git commit的类别，只允许使用下面的标识。

feat：新功能（feature）。

fix/to：修复bug，可以是QA发现的BUG，也可以是研发自己发现的BUG。

- fix：产生diff并自动修复此问题。适合于一次提交直接修复问题
- to：只产生diff不自动修复此问题。适合于多次提交。最终修复问题提交时使用fix

docs：文档（documentation）。

style：格式（不影响代码运行的变动）。

refactor：重构（即不是新增功能，也不是修改bug的代码变动）。

perf：优化相关，比如提升性能、体验。

test：增加测试。

chore：构建过程或辅助工具的变动。

revert：回滚到上一个版本。

merge：代码合并。

sync：同步主线或分支的Bug。

**scope(可选)**

scope用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。

例如在Angular，可以是location，browser，compile，compile，rootScope， ngHref，ngClick，ngView等。如果你的修改影响了不止一个scope，你可以使用*代替。

**subject(必须)**

subject是commit目的的简短描述，不超过50个字符。

建议使用中文（感觉中国人用中文描述问题能更清楚一些）。

- 结尾不加句号或其他标点符号。
- 根据以上规范git commit message将是如下的格式：

```text
fix(DAO):用户查询缺少username属性 
feat(Controller):用户查询接口开发
```

## 9. git stash用法

作用：Stash the changes in a dirty working directory away，通俗来说就是将本地暂时不想提交的修改先临时存储起来，特别在当前已经做了改动但是不想add到暂存区但是又要checkout到其他分支的时候（会消除当前修改）使用

使用方式：`git stash save <自定义名字>`，类似于`git add xxx`但是存储不会放在同一个地方，相互不会影响

`git stash`和`git add`是两个不同的Git命令，它们的作用和使用方式有所不同。

1. `git stash`命令用于保存当前工作目录中的未提交的更改，以便您可以切换到其他分支或恢复到干净的工作目录状态。它的主要作用是将未提交的更改暂存起来，以便稍后恢复。`git stash`命令将未提交的更改保存为一个栈中的存储项，可以在需要时恢复。该命令通常用于**临时保存更改**，并在**切换分支或修复错误时使用**。

   例如：
   ```
   git stash save "my changes"    // 将未提交的更改保存到stash中
   git stash list                 // 列出所有的stash存储项
   git stash apply stash@{0}      // 恢复指定的stash存储项
   ```

2. `git add`命令用于将工作目录中的更改添加到Git的暂存区（也称为索引）。这意味着您想要将这些更改包含在下一次提交中。通过运行`git add`命令，您可以选择性地将文件或更改添加到暂存区，然后可以使用`git commit`命令提交这些更改到版本库。

## 10. 已经add后希望回滚

撤销add进入暂存区的数据：

```shell
git rm --cached <文件名>
```

![image-20230706152215983](http://xwjpics.gumptlu.work/image-20230706152215983.png)

可以看到目前的状态是删除状态，这样操作之后你的修改（例如上面的main.cpp文件）只是在本地文件上的修改，没有add到暂存区

