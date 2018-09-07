title: hexo高级玩法
author: alben.wong
abbrlink: be8242cc
tags:
  - hexo
categories:
  - hexo
keywords: hexo 高级
description: 本文教大家配置一些hexo的高级功能、包括字数统计，阅读统计、评论功能、站内搜索、如何被google和百度的收录、添加留言板，使你的hexo博客显得更加丰富多彩。
date: 2018-09-07 11:15:00
---
## 概要 
添加一些高级功能，可以让我们的网站显得更加丰富，多样性，简单说就是更高逼格。 

## 高级功能
### 文章阅读统计
我选择了 LeanCloud，这也是官方推荐使用的。网上有很多选择不蒜子的，也是可以的。
#### 注册 LeanCloud 账号
[Leancloud官网](https://leancloud.cn/)

#### 创建一个应用 
名字你喜欢就行

#### 创建一个Class
点击进去应用
<img src="/images/hexo高级玩法__0.png" width="500px" height="200px">

注意名字必须为 Counter，勾选无限制的权限。

<img src="/images/hexo高级玩法__1.png" width="400px" height="250px">

#### 修改主题配置
修改 next 主题的_config.yml ，找到 leancloud_visitors ，修改为
```
leancloud_visitors:
  enable: true
  app_id: xxx
  app_key: xxx
```

其中  app_id 和 app_key 在 LeanCloud 的设置 -> 应用 Key 可以找到

<img src="/images/hexo高级玩法__2.png" width="400px" height="250px">

#### 重启查看
这样就配置好了，重新生成 hexo 并发布，我们就可以看到文章阅读次数的统计。  
需要特别说明的是：记录文章访问量的唯一标识符是文章的发布日期以及文章的标题，因此请确保这两个数值组合的唯一性，如果你更改了这两个数值，会造成文章阅读数值的清零重计。

<img src="/images/hexo高级玩法__3.png" width="400px" height="250px">

在 LeanCloud 的后台我们可以看到一个整体的统计量，其中 time 字段就是统计数字，可以修改的哦。

<img src="/images/hexo高级玩法__4.png" width="500px" height="250px">

#### 安全
因为AppID以及AppKey是暴露在外的，因此如果一些别用用心之人知道了之后用于其它目的是得不偿失的，为了确保只用于我们自己的博客，建议开启Web安全选项，这样就只能通过我们自己的域名才有权访问后台的数据了，可以进一步提升安全性。

<img src="/images/hexo高级玩法__5.png" width="450px" height="250px">

### 评论功能
在网上找了很多，有多说，畅言，来必力，gticomment，valine，选择 valine 是因为搞阅读统计的时候已经注册了 LeanCloud，可以顺手用上，而且 next 已经支持了 valine，可以简单快速用起来。

#### 在 LeanCloud 注册和创建应用
上面已经做了

#### 修改主题配置文件
找到 valine 配置项打开 enable，输入 appid 和 appkey ，其他自己设置。
```
valine:
  enable: true
  appid: xxx
  appkey: xxx
  notify: false # mail notifier , https://github.com/xCss/Valine/wiki
  verify: false # Verification code
  placeholder: 走过路过，不留下点什么吗？ # comment box placeholder
  avatar: mm # gravatar style
  guest_info: nick,mail,link # custom comment header
  pageSize: 10 # pagination size
```

#### 效果
这样就我们已经配置好了，重启hexo，看到文章底部出现评论框

<img src="/images/hexo高级玩法__6.png" width="450px" height="250px">

#### 测试一下
hexo d 发布后测试评论一条

<img src="/images/hexo高级玩法__7.png" width="500px" height="250px">

然后在 LeanCloud 后台可以看到，可以进行删除等操作。

<img src="/images/hexo高级玩法__8.png" width="550px" height="250px">


#### 关闭评论
如需取消某个 页面/文章 的评论，在 md 文件的 front-matter 中增加 
```
comments: false
```


### 字数统计
使用 hexo-wordcount 插件，因为 next 主题已经支持了

#### 在 hexo 目录执行安装
```
npm i --save hexo-wordcount
```

#### 修改主题配置
找到 post_wordcount 项
```
post_wordcount:
  item_text: true
  wordcount: true         # 单篇 字数统计
  min2read: true          # 单篇 阅读时长
  totalcount: false       # 网站 字数统计
  separated_meta: true
```

#### 修改显示文字
字数统计和阅读时长是没有单位，需要补上才比较清晰。
修改以下文件
```
themes/next/layout/_macro/post.swig

```
修改【字数统计】，找到如下代码：
```
<span title="{{ __('post.wordcount') }}">
    {{ wordcount(post.content) }}
</span>
```
修改后为：

```
<span title="{{ __('post.wordcount') }}">
    {{ wordcount(post.content) }} 字
</span>
```
同理，我们修改【阅读时长】，修改后如下：
```
<span title="{{ __('post.min2read') }}">
    {{ min2read(post.content) }} 分钟
</span>
```
修改完成后，重新执行启动服务预览就可以了。如下：

<img src="/images/hexo高级玩法__9.png" width="400px" height="250px">

这个阅读的速度是可以修改的，默认是中文300，英文160字/每分钟，详细可以看 [hexo-wordcount](https://github.com/willin/hexo-wordcount)。

### 站内搜索
就是可以在你的网站搜索你网站内的内容
#### 安装 hexo-generator-searchdb
在站点的根目录下执行
```
npm install hexo-generator-searchdb --save
```
#### 修改站点配置文件
添加
```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
#### 修改 next 主题配置文件
找到 local_search 修改为 true 
```
local_search:
  enable: true
```
#### 效果
重启 hexo，可以看到在目录栏最下方出现了“搜索”菜单

<img src="/images/hexo高级玩法__10.png" width="250px" height="200px">

点击弹出框，就可以搜索
<img src="/images/hexo高级玩法__11.png" width="450px" height="250px">

### 被Google和百度收录
#### google
##### 登陆 google search console
[google search console](https://search.google.com/search-console/welcome) 

添加你的网站地址（需要google账号）

<img src="/images/hexo高级玩法__12.png" width="450px" height="250px">

##### 进行验证
google 需要验证你拥有该网站的权限，默认推荐的验证方式是在你的网站添加一个它提供的 html，但是由于 hexo 的静态文件是
生成的，我们 clean 之后就没了，所以我们不适用这种方式（其实也可以做到）。我们使用另一种更加方便的方式。
使用 meta 标签

<img src="/images/hexo高级玩法__13.png" width="450px" height="250px">

做法是修改主题配置文件，找到 google_site_verification，值修改为 google 提供的 meta 中 content 的内容

```
google_site_verification: xxxxx
```

加了这个配置后 next 会自动帮我们插入 meta 标签了。
我们重启，发布。然后点击上图的验证按钮，成功的话，就会看到以下提示

<img src="/images/hexo高级玩法__14.png" width="450px" height="250px">

然后我们点击“前往资源页面”，对我们网站其中一个页面进行检查，会提示站点不适用

<img src="/images/hexo高级玩法__15.png" width="450px" height="250px">

##### 增加站点地图
安装插件
```
npm install hexo-generator-sitemap --save
```

在站点配置文件添加
```
sitemap:
  path: sitemap.xml
```

修改站点配置文件，找到 url 项，改为你网站地址。默认是
```
http://yoursite.com
```
如果你不修改这个，sitemap.xml 生成内容不正确。

```
url: https://albenw.github.io
```

重新生成、发布，可以看到在 public 目录下生成了 sitemap.xml 文件。
在 google search console 提交站点地图

<img src="/images/hexo高级玩法__16.png" width="550px" height="250px">

提交后结果，看到成功的状态

<img src="/images/hexo高级玩法__17.png" width="550px" height="250px">

在覆盖率可以看到 google 抓取你的页面，但是我们刚刚添加的网站还没被抓取，要等搜索引擎下一次更新索引你才能在 google 上搜到，请耐性等待。

<img src="/images/hexo高级玩法__18.png" width="550px" height="250px">

##### 添加 robot.txt
原来我是漏掉这一步，了解后发现原来这个文件对爬虫的抓取有一定的帮助，这也是SEO的优化，所以加上。
在站点 source 目录下创建 robots.txt，内容如下：

```
# hexo robots.txt
User-agent: *
Allow: /
Allow: /archives/

Disallow: /vendors/
Disallow: /js/
Disallow: /css/
Disallow: /fonts/
Disallow: /vendors/
Disallow: /fancybox/

Sitemap: https://albenw.github.io/sitemap.xml
Sitemap: https://albenw.github.io/baidusitemap.xml
```

#### baidu

#### 打开百度的站点管理
[百度站点管理](https://ziyuan.baidu.com/site)

添加一个站点

<img src="/images/hexo高级玩法__19.png" width="550px" height="250px">

#### 验证
同样我们使用 meta 标签验证

<img src="/images/hexo高级玩法__20.png" width="550px" height="250px">

修改主题配置文件，添加 baidu_site_verification 项，值为 content 内容。
```
baidu_site_verification: xxx
```

注意，原来配置文件是没有 baidu_site_verification 这个项的，但是通过查看 layout/\_partials/head.swig，我们发现其实 hexo 是支持的，如果 head.swig 没有，则需要我们手动在 head.swig 增加

```
{% if theme.baidu_site_verification %}
  <meta name="baidu-site-verification" content="{{ theme.baidu_site_verification }}" />
{% endif %}
```
配置好后，重新生成，发布，在点击百度站点页面的验证按钮。


#### 主动推送
由于 github 禁止了百度的爬虫，所以我们不能像对 google 那样通过 sitemap 的方式被抓取到链接，即使你配置了也是没用的。  
除了 sitemap 还有主动推动和自动推送这两种方式，主动推动的原理是每次 deploy 的时候都把所有链接推送给百度，自动则是每次网站被访问时都把该链接推送给百度。
通过对比，我觉得主动推动比价好，所以选用这种方式

##### 插件安装
```
npm install hexo-baidu-url-submit --save
```

##### 修改站点配置文件
在 \_config.yml，添加以下内容
```
baidu_url_submit:
  count: 5
  host: your_site
  token: your_token
  path: baidu_urls.txt 
```

其中 count 表示一次推送提交最新的N个链接；host 和 token 可以在百度站点页面->数据引入->链接提交可以找到；path 为生成的文件名，里面存有推送的，我们网站的链接地址。

<img src="/images/hexo高级玩法__21.png" width="550px" height="250px">

确保站点配置文件中的 url 项跟百度注册的站点一致

同样修改站点配置文件的 deploy 项，我们原来已经有 git 的 deploy，现在增加对 baidu 的推送，最终是这样子的

```
deploy:
- type: git
  repo: git@github.com:albenw/albenw.github.io.git
  branch: master
- type: baidu_url_submitter
```

重新生成，发布 hexo d，可以看到推送给百度成功

<img src="/images/hexo高级玩法__22.png" width="450px" height="250px">

我们可以在百度站点页面->数据引入->链接提交看到成交推送的链接数量，不过还不能看当天的，要等明天。

（一天后）可以看到提交量了。

<img src="/images/hexo高级玩法__23.png" width="450px" height="250px">

虽然推送成功了，但是百度不是马上抓取的，需要耐心等待，具体可以查看数据监控->索引量页面。

### 留言板
所谓留言板其实就是开一个空的 page ，然后可以有评论这样子。

#### 添加留言板的 page
```
hexo new page guestbook
```

#### 修改主题配置文件
找到 next 的 \_config.yml  文件里面的 menu 项，增加
```
guestbook: /guestbook
```

因为这里使用的是中文，找到 next 主题的 languages 目录里面的zh-Hans.yml文件，menu子项中添加

```
guestbook: 留言
```

#### 设置留言板的图标
next 主题的默认是 page 的名字就是图标 icon 的名字，由于没有 "guestbook" 这个 icon ，所以留言板左边的小图标是一个问号。
由于 next 支持 Font Awesome 的所有图标，所以只需要到 Font Awesome 的网站找到你想要的图标，然后还是在主题配置文件的 menu 项，最终修改为

```
guestbook: /guestbook || comment 
```

#### 效果
page 默认是开了 comments 的，所以直接用就可以了。

<img src="/images/hexo高级玩法__24.png" width="450px" height="250px">


### 参考资料
[为NexT主题添加文章阅读量统计功能](https://notes.doublemine.me/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#%E9%85%8D%E7%BD%AELeanCloud)
https://blog.csdn.net/blue_zy/article/details/79071414
https://www.jianshu.com/p/baea8c95e39b  
http://theme-next.iissnan.com/third-party-services.html#local-search
https://www.jianshu.com/p/25145964abf3
https://www.jianshu.com/p/5e68f78c7791  
[Hexo插件之百度主动提交链接](https://hui-wang.info/2016/10/23/Hexo%E6%8F%92%E4%BB%B6%E4%B9%8B%E7%99%BE%E5%BA%A6%E4%B8%BB%E5%8A%A8%E6%8F%90%E4%BA%A4%E9%93%BE%E6%8E%A5/)
[Hexo博客提交百度和Google收录](http://fengdi.org/2017/08/10/Hexo%E5%8D%9A%E5%AE%A2%E6%8F%90%E4%BA%A4%E7%99%BE%E5%BA%A6%E5%92%8CGoogle%E6%94%B6%E5%BD%95.html)
