---
author: "Suika"
title: "tar.xz解压"
date: "2020-03-16 11:08:56"
description: ""
categories: "Tech"
tags: 
  - "tar"
image: ""
---

遇到个包xx.tar.xz打包的，感觉这格式很罕见，有点懵，所以查了一下要怎么解压，顺便把tar复习一下。


## tar.xz解压：

```bash
    # yum install -y xz
    # xz -d ***.tar.xz
    # tar -xvf  ***.tar
```

这种格式本质是打包了两层

也可以不装xz，直接使用 tar xvJf  **.tar.xz来解压

## Tar选项：
{{< highlight bash >}}
    c – 创建压缩文件
    x – 解压文件
    v – 显示进度.
    f – 文件名.
    t – 查看压缩文件内容.
    j – 通过bzip2归档
    z –通过gzip归档
    r – 在压缩文件中追加文件或目录
    W – 验证压缩文件
{{< /highlight >}}