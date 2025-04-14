---
title: 语言Markdown
date: 2023-10-30 00:00:00
updated: 2023-10-30 00:00:00
description: Markdown是一种轻量级的标记语言，其简化了繁琐的文章格式调整，并提供了简介的图像表示方法
categories:
- 知识
- 语言
- 标记语言
- Markdown
tags:
- Markdown
aside: true
top_img: img/a4.png
cover: img/a4.png
---

# 什么是Markdown

Markdown是一种纯文本格式的标记语言。通过简单的标记语法，它可以使普通文本内容具有一定的格式。

# 如何书写Markdown语言的文件

使用Typora可以书写Markdown语言的文件，其支持即时编辑，并可以在代码和展示间切换。

# 语法说明

在实际使用过程中，似乎不同环境下有细节上的不同，比如在VSCODE下书写表格前需要换行，而在Typora里不用。

## 文字相关

### 标题

使用 {% label '\# 标题' %}，表示标题，根据标题前的\#数量表示标题等级，注意**空格**。
文章的TOC会根据标题生成，最多六级。由于会破坏TOC格式，标题展示放在了文章最后。

### 字体

* 加粗
使用 {% label '\*\*文字\*\*' %}  ，表示加粗对应文字，注意**没有空格**。
* 斜体
使用  {% label '\*文字\*' %} ，表示倾斜对应文字，注意**没有空格**。
* 斜体加粗
使用 {% label '\*\*\*文字\*\*\*' %} ，表示斜体加粗对应文字，注意**没有空格**。
* 删除线
使用 {% label '\~\~文字\~\~' %} ，表示为对应文字加删除线，注意**没有空格**。

>**加粗**
>*斜体*
>***斜体加粗***
>~~删除~~

### 引用

使用{% label '\>引用' %}  ，表示引用，根据引用前的\>数量表示引用等级。

> 一级引用
>>二级引用
>>>三级引用
>>>>>>>>多级引用

### 代码

使用{% label ' \`代码\`' %} ，表示代码。
使用 {% label '首行 \`\`\`(语言说明) &emsp;&emsp; 多行代码 &emsp;&emsp;  尾行\`\`\`' %} ，表示代码块。

>`printf("代码");`
>```C
>void f()
>{
>    for(int i=1;i<=n;i++)
>     printf("代码块");
>}
>```
### 数学公式

略

### 超链接

使用{% label '\[超链接名\]\(超链接地址 "超链接Tittle"\)' %} ，表示超链接，Tittle可不写。

>[我的博客](https://VVolfBite.github.io)

## 图形相关

### 分割线

使用{% label '---' %}，表示分割线。

>---

### 列表

#### 无序列表

使用{% label '* 列表内容' %}，表示无序列表项，注意**空格**。

>* 第一项
>* 第二项
>* 第三项

#### 有序列表

使用{% label 'number. 列表内容' %}，表示有序列表项目。

>1. 第一项
>2. 第二项
>3. 第三项

### 表格

使用 {% label '| 表头1 | 表头2 | .... | 表项n |' %}，表示表头。
使用 {% label '| ---- | ---- | ... | ---- |' %},标志表头表格内容分界线。
使用 {% label '| 表项1 | 表项2 | ... | 表项n |' %}，表示表格内容。

> | 表头1 | 表头2 | 表头3 |
> | ----  | ----  | ----  |
> | 1行1列 | 1行2列 | 1行3列 |
> | 2行1列 | 2行2列 | 2行3列 |
> | 3行1列 | 3行2列 | 3行3列 |  

### 图片

使用 {% label '\!\[图片说明\]\(图片URL 图片title\)' %},表示引用在URL下的图片。

> ![这是一张图片](https://www.giaott.com/image/2sAp9)

## 其他内容

### 内嵌HTML

使用{% label 'center标签' %}，表示居中。
><center>居中</center>

使用{% label 'br标签' %}，表示换行。
>换 <br> 行

使用{% label '&+ensp;' %}，表示半角空格。
>空 &ensp; 格

使用{% label '&+emsp;' %}，表示全角空格。
>空&emsp; 格

使用{% label 'font标签+face(字体)/Size(字号)' %}，表示字体信息。
> <font face="华文行楷">华文行楷</font>
> <font Size=1>1号字体</font>
> <font Size=5>5号字体</font>

### HEXO-Butterfly主题下的特殊用法 标签外挂

标签外挂是指将标签添加到文章文字前。
一般格式为{% label '\{\% 部件 配置说明1&ensp;配置说明2&ensp; ... &ensp;配置说明n \%\}' %},需要多行描述的结尾需要书写{% label '\{\%end部件\%\}' %}。
注意相关空格。
#### 高亮label标签

部件为{% label label %}，配置如下：

| 名称 | 用法 |
| ----  | ---- |
| text	| 文字 |
| color	| 【可选】背景颜色，默认为 default/default/blue/pink/red/purple/orange/green |

>{% label label %}

#### 笔记note标签

部件为{% label note %}，配置如下：

| 名称 | 用法 |
| ----  | ---- |
| class | 【可选】标识，不同的标识有不同的配色（ default / primary / success / info / warning / danger ）|
| icon | 【可选】可配置自定义 icon (只支持 fontawesome 图标, 也可以配置 no-icon ) |
| style | 【可选】可以覆盖配置中的 style（simple/modern/flat/disabled）|

需要使用{% label '\{ \%endnote\% \}' %}结尾。

>{% note red 'fas fa-fan' simple%}
演示内容1
{% endnote %}
>{% note warning simple %}
演示内容2
{% endnote %}

#### 隐藏hide标签

部件为{% label hide+Inline/Block/Toggle %}，其中不能含有 **英文逗号** ，可以使用 **&sbquo;** ，配置如下：
| 名称 | 用法 |
| ----  | ---- |
| content,display,bg,color（Inline） | 【必需】指明隐藏的内容，展示的内容(可选)，背景色(可选)，字体色(可选) |
| display,bg,color（Block/Toggle） | 【必需】展示的内容(可选)，背景色(可选)，字体色(可选) 隐藏的内容在下一行写 |
需要使用{% label '\{ \%endhide+block/Toggle\% \}' %}结尾。

>这是什么类型的hide？{% hideInline Inline,查看答案,#FF7242,#fff %}
>这是什么类型的hide？{% hideBlock 查看答案,#FF7242,#fff %}
Block
{% endhideBlock %}
>这是什么类型的hide？{% hideToggle 查看答案,#FF7242,#fff %}
Toggle
{% endhideToggle %}

#### 表格tabs标签



#### 按钮btn标签

部件为{% label btn %}，配置如下：

| 名称 | 用法 |
| ----  | ---- |
| [url] | 链接 |
| [text] | 按钮文字 |
| [icon] | [可选] 图标 |
| [color] | [可选] 按钮背景顔色(默认style时），按钮字体和边框顔色(outline时)，default/blue/pink/red/purple/orange/green | 
| [style] | [可选] 按钮样式 默认实心，outline/留空 |
| [layout] | [可选] 按钮佈局 默认为line，block/留空 |
| [position] | [可选] 按钮位置 前提是设置了layout为block 默认为左边，center/right/留空 |
| [size] | [可选] 按钮大小，larger/留空 |
>我的博客 {% btn 'https://VVolfBite.github.io',Butterfly,far fa-hand-point-right,outline %}





#### 图片群galleryGroup标签

需要使用{% label '\< \div \class\=\"gallery-group-main\">' %}开始。

部件为{% label 'galleryGroup' %}，配置如下：


| 名称 | 用法 |
| ----  | ---- |
| name | 【必需】图库名字 |
| description | 【必需】图库描述 |
| link | 【必需】连接到对应相册的地址|
| img-url | 【必需】图库封面的地址 |

需要使用{% label '\< \/div >' %}结尾。
><div class="gallery-group-main">
>{% galleryGroup '壁纸' '收藏的一些壁纸' 'https://www.giaott.com/image/2sAp9' './img/default.jpg' %}
></div>

#### 图片gallery标签

部件为{% label gallery %}。
中间使用Markdown描述图片。
需要使用{% label '\{ \%endgallery\% \}' %}结尾，配置如下：


>{% gallery %}
![](https://www.giaott.com/images/2022/07/15/2sAp9.md.jpg)
![](https://www.giaott.com/images/2022/07/15/2sAp9.md.jpg)
![](https://www.giaott.com/images/2022/07/15/2sAp9.md.jpg)
![](https://www.giaott.com/images/2022/07/15/2sAp9.md.jpg)
{% endgallery %}

#### 行内图片inlineImg标签

部件为{% label inlineImg %}

| 名称 | 用法 |
| ----  | ---- |
| src | 【必需】图片URL |
| height | 【可选】图片限高，不选则与文字一样高 |

>图{% inlineImg https://www.giaott.com/images/2022/07/15/2sAp9.md.jpg 100px %}片

># 一级标题
>## 二级标题
>### 三级标题
>#### 四级标题
>##### 五级标题
>###### 六级标题

