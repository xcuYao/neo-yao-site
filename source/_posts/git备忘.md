---
title: git备忘
date: 2024-03-22 15:38:47
tags:
---

#### 配置
```
// 修改系统级配置 通常保存在git的安装目录下的/etc/gitconfig
$ git config --system -e
// 修改全局级配置 win保存在C:\Users\xxxx\.gitconfig linux保存在/home/xxx/.gitconfig
$ git config --global -e 
// 修改项目级配置 保存在项目的.git\config中
$ git config --local -e

// 以下是一些常用配置 可以写到配置文件中 
[alias]
    co = checkout
    br = branch
    ci = commit
    #
    st = status
    # log带彩色
    lg = "log --graph --pretty=format:' %C(auto)%h %d%s %Cblue%an %Cgreen%ar'"
    # 删除最近一个提交，保留文件修改
    undo = reset --soft HEAD^
    # 删除最近一个提交，不保留文件
    cancel = reset --hard HEAD^
    # 提交完了，发现还需要一点小修改，不想新建一个提交
    onemore = commit -a --amend --no-edit
[push]
    default = simple
[pull]
    rebase = true

```
#### 易忘命令
```
// 清除本地缓存
$ git rm -r --cached .

----------
// rebase合并多个commit 不包含(HashA)
$ git rebase -i <HashA>
// 或向前三个节点
$ git rebase -i HEAD~3

// 将pick改为squash 或 s 表示该commit和前一个commit合并
// 再保存即可
-----------

// chery-pick 将HashA和B之间的commit转移过来（不包括A）
$ git chery-pick <HashA> <HashB>
// 如果要包含A
$ git chery-pick A~..B

// 分支重命名
git checkout <旧分支名>         # 切换到需要重命名的分支
git branch -m <新分支名>        # 重命名本地分支
git push --delete origin <旧分支名>      # 删除远程旧分支
git branch --unset-upstream      # 解除之前建立的分支关系
git push --set-upstream origin <新分支名> # 推送到远端并建立关系
git push -u origin <新分支名> # 推送到远端并建立关系

// 修剪分支
git remote prune origin --dry-run 
```

