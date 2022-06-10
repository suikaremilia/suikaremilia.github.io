---
author: "Suika"
title: "转到HUGO"
date: "2022-05-25 18:55:10"
description: "以后可以没有客户端了"
categories: 
  - "mumbler"
  - "Tech"
tags: 
  - "mumbler"
image: ""
---

没有谁像我一样吧，Gridea终于更新了却不想用了。  
花了点时间把原来Gridea的站转到了HUGO，体验下持续集成，从此工作写字的地方统一到VS code了，~~然而要手攒FrontMatter了~~。  

挺简单的过程：

## 安装hugo，并下载主题
```bash
scoop install hugo
scoop install hugo-extended

hugo new site my_blog

cd my_blog

git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack

cp -r themes/hugo-theme-stack/assets .
cp themes/hugo-theme-stack/config.yml .
```

## 根据文档调整一下config，写几个yml
https://docs.stack.jimmycai.com/zh/getting-started

https://github.com/CaiJimmy/hugo-theme-stack

## 迁移原来的文字
Gridea的文字在post目录下，hugo在content/post下  
Gridea的图片在post-images目录下，hugo建议在文字的相同目录下

有个脚本可以批量，但看了下我的数量，~~手动也不是啥大工作量~~。  
https://github.com/wherelse/Gridea2Hugo  
然而工具用了却还是手动了一下，毕竟Gridea没有description这字段。

其它还干了啥不记得了，中间时不时的打开`hugo server`本地看看效果。

## 部署到GitHub

建个`username.github.io`的repo，然后除了push还能干啥呢？  
```bash
git init .
git add .
git commit -m "init"
git remote add origin https://github.com/suikaremilia/suikaremilia.github.io.git
git branch -M main
git push -u origin main
```

去github上生成一个token，路径：Settings -> Developer Settings -> Personal access tokens，构选repo和workflow，记得保存token的内容。

然后去`username.github.io`的repo操作Settings -> Secret为仓库添加一个名为`personal_token`的密钥。

之后`pull`一下仓库，在`.github`下建一个`workflows`的文件夹，写一个名为`main.yml`的文件：  
```yaml
name: Deploy Hugo

on:
  push:
    branches:
      - main
      
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: latest

      - name: Build 
        run: hugo

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          personal_token: ${{ secrets.personal_token }}
          PUBLISH_BRANCH: gh-pages
          PUBLISH_DIR: ./public  
          commit_message: ${{ github.event.head_commit.message }}
```

再`commit`一次

再修随便修改个文件，然后再提交一次，使其生效

然后，然后你访问`username.github.io`也只能看到404

还需要到repo的setting里pages设一下Source的branch为`hg-pages`，至此可以访问`username.github.io`

至此就成功了。

## 参考
https://yuukoamamiya.github.io/p/%E4%BB%8Egridea%E8%BF%81%E7%A7%BB%E5%88%B0hugo/#%E8%AE%BE%E7%BD%AEgithub-actions