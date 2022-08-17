---
layout:     post
title:      "Endnote与SCI-Hub联用"
date:       2022-08-17 21:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - Endnote X20
    - sci-hub
---

## Endnote X20的安装以及与scihub的联用  

### 1.下载安装
[下载地址:Endnote20.rar](https://mailustceducn-my.sharepoint.com/:u:/g/personal/xh125_mail_ustc_edu_cn/EQkmBvVWKetIvf7eAu8mwVoBwT4LKkVehgB2MUVDR2YooA?e=KphC7g)

**安装说明**：EndNote软件为正版软件，正确安装时不存在试用期和需要产品号的问题。下载软件后请完全解压缩到一个新文件夹内进行安装（不要直接双击打开）。压缩后会生成 EN20Inst.msi、License.dat两个文件到同一个文件夹中，之后双击EN20Inst.msi文件进行安装，不需要输入序列号。

### 2.Endnote与Sci-hub联用
简介：利用EndNote中存储的DOI信息，通过设置新的样式，添加可用的SCI-hub官方网页前缀生成相应网页链接，之后则可以进行下载，效果如下：点击链接则可直接下载文献
![scihub.png](https://xh125.github.io/images/post/scihub.png)

### 3.样式设置步骤

1. 打开Tools-设置output style-选择New style
![new style](https://xh125.github.io/images/post/scihub-1.png)
2. 选择Bibliography - Templates-输入最近可用的sci-hub官网并添加//DOI
![doi](https://xh125.github.io/images/post/scihub-2.png)  
3. 设置样式的名字并保存，保存后选择刚刚设置好的style（例如将样式名字设置为sci-hub）  

4. SCI-hub官方网站由于其特殊性，老是在改变，每次链接不可用时只需修改网址即可。目前可用的scihub镜像包括：

```php
https://sci-hub.se/DOI
https://sci-hub.ru/DOI
https://sci-hub.st/DOI
https://sci-hub.ee/DOI
```
