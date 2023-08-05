---
author: "Suika"
title: "Vim替换"
date: "2020-03-20 10:05:31"
description: "VIM是最好的编辑器"
categories: 
  - "Tech"
tags: 
  - "vim"
image: ""
---

Vim可以在命令模式下使用substitute命令，将指定的字符串替换成目标字符串。
## 基本语法：
Vim替换命令的基本语法是 :[range]s/目标字符串/替换字符串/[option]，其中range和option字段都可以缺省不填。

**range:** 表示搜索范围，默认表示当前行;
也可以是以逗号隔开的两个数字，例如1,20表示从第1到第20行;
也可以是特殊字符，与其他地方的默认习俗一致，其中%表示整个文件

**s** 即substitute的简写，表示执行替换字符串操作;

**option:** 表示操作类型，默认只对第一个匹配的字符进行替换；
常见字段值有：g(global)表示全局替换;
c(comfirm)表示操作时需要确认;
i(ignorecase)表示不区分大小写;

## 简单实例
**替换当前行第一个 VIM 为 sakuya**
`:s/VIM/sakuya/`

**替换当前行所有 VIM 为 sakuya**
`:s/VIM/sakuya/g `

**替换第 n 行开始到最后一行中每一行的第一个 VIM 为 sakuya**
`:n,$s/VIM/sakuya/ `

**替换第 n 行开始到最后一行中每一行所有 VIM 为 sakuya**

`:n,$s/VIM/sakuya/g `
**替换每一行的第一个 VIM 为 sakuya**

`:%s/VIM/sakuya/`
等同于：
`:g/VIM/s//sakuya/`

**替换每一行中所有 VIM 为 sakuya**
`:%s/VIM/sakuya/g`
等同于： 
`:g/VIM/s//sakuya/g`

**可以使用 # 作为分隔符，此时中间出现的 / 不会作为分隔符，例如：替换当前行第一个 VIM/ 为 sakuya/**
`:s#VIM/#sakuya/# `

**全文的行首加入//字符，批量注释时非常有用，/进行转义后才可以替换**
``
:%s/^/\/\//
``

**将所有行尾多余的空格删除**
`:%s= *$==`
