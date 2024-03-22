---
title: win备忘
date: 2024-03-22 16:15:33
tags:
---

#### powershell
``` bash
// 窗口中打开当前路径 以下三个命令均可
$ explorer .
$ start .
$ ii .

// 获取当前有效的执行策略
$ Get-ExecutionPolicy

// 修改策略
$ Set-ExecutionPolicy RemoteSigned
```
一般执行`Get-ExecutionPolicy`为`Restricted`,表示为 允许单个命令，但不允许脚本;
这也是为什么执行node相关脚本报错的原因，修改为`RemoteSigned`，脚本即可运行。
具体可以参考[PowerShell 执行策略](https://learn.microsoft.com/zh-cn/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.4)  