title: hexo优化
author: alben.wong
abbrlink: 3460d887
tags:
  - hexo
categories:
  - hexo
keywords: hexo优化
description: 对hexo的一些优化，有图片的插入和存放，文件压缩，永久link，预览，SEO优化，还包括一些使用上的小技巧，能够为hexo的日常使用提高性能和便捷性。
date: 2018-09-06 17:12:00
---
## 概要
按照我之前hexo的安装部署，可以正常使用，但是或者存在性能或效率的问题，又或者在操作上不便，这篇文章希望能做一点优化和改善。

## 优化
### 图片插入与存放问题
一般来说有一下两种方式
#### 图床
就是图片的云存储，图片存放在云上，这种方式一般是先把图片上传上去，获取到链接，然后在 MD 中引用。  
我个人觉得这种做法操作麻烦，使用图床麻烦，要先上传图片又麻烦，而且如果图床不稳定，你的图片就可能显示不出来了，甚至图床挂了，你的图片就没了。

#### 本地
可能我习惯用云笔记，我个人偏向使用在本地的。本地的做法一般是先把图片放在 hexo 站点的目录下，然后在 MD 中引用，这样也可以把图片上传到 github 做备份保存。  
但我觉得还是有两个问题，一是操作麻烦，二是管理图片。第一，我个人喜欢用 hexo-admin，直接在页面复制图片就行了。
第二，hexo-admin 默认是放在 images 目录下的，但是如果文章越来越多，图片会很乱。关于这点 hexo 提供 post_asset_folder 参数配置，为 true 的话，在新建 post 时会在 \_post 建一个同名文件夹（仅此而已），hexo 的初衷是想我们把图片放在里面，可惜 hexo-admin 对这个配置还不支持，我看它还是在 issue 里。  
所以到这里，我觉得还是没有好的做法，我自己的做法是放 images 目录，图片的命名要有规范，例如 post_name + "\_\_" + index 这样，方便做管理。

这里提醒大家一点，编辑时使用的图片的路径和生成 html 时的是不一样的。

### html，js，css，images 压缩
使用 hexo-all-minifier 插件，在站点目录执行
```
npm install hexo-all-minifier --save
```
在站点配置文件 _config.yml ，增加一行即可
```
all_minifier: true 
```

在 hexo g 生成的时候会看到打印输出 xx% saved 这样的字眼，表示成功了。我觉得感觉是好像是。。快了一点。。吧。。
<img src="/images/hexo优化__0.png" width="500px" height="200px">

### 文章唯一link
更改文章题目或者变更文章发布时间，在默认设置下，文章链接都会改变，不利于搜索引擎收录，也不利于分享。这里还是涉及爬虫的知识点，如果链接的层级太深，则对SEO不友好。所以简短的、唯一永久链接才是更好的选择。

#### 安装插件
```
npm install hexo-abbrlink --save
```
在站点配置文件中查找代码permalink，将其更改为
```
permalink: posts/:abbrlink/  # “posts/” 可自行更换
```

#### 修改配置
然后在站点配置文件中添加如下代码
```
# abbrlink config
abbrlink:
  alg: crc32  # 算法：crc16(default) and crc32
  rep: hex    # 进制：dec(default) and hex
```

#### 效果
重启 hexo 生效后，可以看到文章的链接不再是“日期+文章名”，而是配置的 permalink，后面的一串字符就是 abbrlink。

<img src="/images/hexo优化__1.png" width="400px" height="100px">

特别的说明：由于加了这个配置之后文章的链接URL变了，所以之前如果有做“评论”或“访问计数”配置的，就会全部失效。

### 预览
首页进去是对每一篇文章都显示了所有内容，需要把当前文章滚动到末尾才能看到下一篇文章，这样不能让读者快速浏览到大概有哪些文章，不能一下子吸引到读者。  
在主题配置文件中找到 auto_excerpt 属性，将enable设置为true ，将length设置为想要预览到的字数
```
auto_excerpt:
enable: true #将原有的false改为true
length: 300  #设置预览的字数
```

在首页看到的效果图，它的摘要只是把文本存粹的按照 length 截取出来。

<img src="/images/hexo优化__2.png" width="400px" height="100px">

### SEO优化
做seo优化有利于搜索引擎对你网站的索引，根据关键字提高你网站的排名，提高曝光率。

#### title 优化
使首页改为“网站名称-网站描述”这样的显示方式。
##### 打开 seo 项
在主题配置文件找到 seo 项
```
seo: true
```

##### 修改 post 模版
在站点目录 scaffolds\post.md 文件，添加 keywords 和 description 字段，用于生成的文章中添加关键字和描述。
```
title: {{ title }}
date: {{ date }}
tags:
keywords:
description:
---
```
这样在首页文章的预览中就会变成 description，利于 SEO。

<img src="/images/hexo优化__3.png" width="400px" height="100px">

#### 添加 “nofollow” 标签
>nofollow是HTML的一个属性，用于告诉搜索引擎不要追踪特定的网页链接。可以用于阻止在PR值高的网站上以留言等方式添加链接从而提高自身网站排名的行为，以改善搜索结果的质量，防止垃圾链接的蔓延。网站站长也可对其网页中的付费链接使用nofollow来防止该链接降低搜索排名。对一些重要度低的网页内容使用nofollow，还可以使搜索引擎以不同的优先级别来抓取网页内容。

by 维基百科

##### 修改footer.swig文件
在 next 目录 layout\_partials，找到两处 a标签加上 rel="external nofollow" 属性。

```
{{ __('footer.powered', '<a rel="external nofollow" class="theme-link" target="_blank" href="https://hexo.io">Hexo</a>') }}
```

```
<a rel="external nofollow" class="theme-link" target="_blank" href="https://github.com/iissnan/hexo-theme-next">
```

##### 修改sidebar.swig文件
在 next 目录 layout\_macro，将下面代码中的a标签加上rel="external nofollow"属性，顺序如下。

```
<a rel="external nofollow" href="{{ link.split('||')[0] | trim }}" target="_blank" title="{{ name }}">
```
```
<a href="https://creativecommons.org/{% if theme.creative_commons === 'zero' %}publicdomain/zero/1.0{% else %}licenses/{{ theme.creative_commons }}/4.0{% endif %}/" rel="external nofollow" class="cc-opacity" target="_blank">
```
```
<a href="{{ link }}" title="{{ name }}" rel="external nofollow" target="_blank">{{ name }}</a>
```

其实就是把一些含有 target="_blank" 或 链去其他网站的超链接给加上 nofollow ，提升 SEO 效率。

#### 唯一链接 permalink
这个我们在上面已经做了。

## 小技巧
### 文章内引用自己的文章

这是hexo的标签语法
```
{% post_link 文章文件名（不要后缀） 文章标题（可选） %}

{% post_link Hello-World %}
{% post_link Hello-World 你好世界 %}

```
~~注意文章名字和文章标题不能有空格，有的话不能生效，还不知怎么解决。~~

直到我增加了站内搜索功能后，好奇搜索结果是怎么链接到文章的呢，于是我看了一下，如下
```
<a href="/2018/09/04/Hexo-Github-Pages安装部署/" class="search-result-title"><b class="search-keyword">Hexo</b>+Github Pages安装部署</a>
```
可以看出原来 post 的名字是 “Hexo+Github Pages安装部署”，但是生成静态页面就变成了 “Hexo-Github-Pages安装部署”。然后我拿这个 “Hexo-Github-Pages安装部署”放到上面一试，发现可以了！
看来生成后 post 的名字如果单词之间有特殊符号会统一变成“-”？？

### 插入图片
在 hexo-admin 直接复制图片会是这样子
```
![upload successful](/images/hexo优化__0.png)
```
但是这样直接显示在页面不适合，我们一般需要调整大小或位置

#### 调整图片的显示
##### hexo 支持的标签语法
```
{% img [class names] /path/to/image [width] [height] [title text [alt text]] %}
```
不过不能在 hexo-amdin 看到。

##### img 标签
```
<img src="/images/hexo-admin安装使用__0.png" width="600px" height="200px" align=center>
```
这样可以在直接 hexo-admin 中显示，路径也兼容生成后的 html。