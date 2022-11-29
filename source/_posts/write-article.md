---
title: 怎样使用hexo编写文章 # 自动创建，如 hello-world
date: 2019-10-29 01:47:21 # 自动创建，如 2019-09-22 01:47:21
tags:
  - HowToWriteMarkDownArticle
categories:
  - tools
---

## 创建 MD 文章

```bash
$ hexo new "My New Post"
```

创建的文章会出现在 source/\_posts 文件夹下，是 MarkDown 格式。  
在文章开头通过如下格式添加必要信息：

```
---
title: 标题 # 自动创建，如 hello-world
date: 日期 # 自动创建，如 2019-09-22 01:47:21
tags:
- 标签1
- 标签2
- 标签3
categories:
- 分类1
- 分类2
---
```

这样在下次编译的时候就会自动识别标题、时间、类别等等，另外还有其他的一些参数设置，可以[参考文档](https://hexo.io/zh-cn/docs/writing.html)

## 标签页

我们的博客只有首页、文章页，如果我们想要增加标签页，可以自行添加，这里 Hexo 也给我们提供了这个功能，在根目录执行命令如下：

```
hexo new page tags

```

执行这个命令之后会自动帮我们生成一个 source/tags/index.md 文件，内容就只有这样子的：

```
---
title: tags
date: 2019-09-26 16:44:17
---
```

我们可以自行添加一个 type 字段来指定页面的类型：

```
type: tags
comments: false
```

然后再在主题的 \_config.yml 文件将这个页面的链接添加到主菜单里面，修改 menu 字段如下：

```
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  #categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

## 分类页

分类功能和标签类似，一个文章可以对应某个分类，如果要增加分类页面可以使用如下命令创建：

```
hexo new page categories
```

然后同样地，会生成一个 source/categories/index.md 文件。

我们可以自行添加一个 type 字段来指定页面的类型：

```
type: categories
comments: false
```

然后再在主题的 \_config.yml 文件将这个页面的链接添加到主菜单里面，修改 menu 字段如下：

```
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

## 搜索页

很多情况下我们需要搜索全站的内容，所以一个搜索功能的支持也是很有必要的。

如果要添加搜索的支持，需要先安装一个插件，叫做 hexo-generator-searchdb，命令如下：

```
npm install hexo-generator-searchdb --save
```

然后在项目的 \_config.yml 里面添加搜索设置如下：

```

# Local search
# Dependencies: https://github.com/wzpan/hexo-generator-search
local_search:
  enable: true
  # If auto, trigger search by changing input.
  # If manual, trigger search by pressing enter key or search button.
  trigger: auto
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 5
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false
```

这里用的是 Local Search，如果想启用其他是 Search Service 的话可以参考官方文档：https://theme-next.org/docs/third-party-services/search-services。

## 404 页面

另外还需要添加一个 404 页面，直接在根目录 source 文件夹新建一个 404.md 文件即可，内容可以仿照如下：

```

---
title: 404 Not Found
date: 2019-09-22 10:41:27
---

<center>
对不起，您所访问的页面不存在或者已删除。
您可以<a href="https://blog.nightteam.cn>">点击此处</a>返回首页。
</center>

<blockquote class="blockquote-center">
    NightTeam
</blockquote>
```

这里面的一些相关信息和链接可以替换成自己的。

增加了这个 404 页面之后就可以

完成了上面的配置基本就完成了大半了，其实 Hexo 还有很多很多功能，这里就介绍不过来了，大家可以直接参考官方文档：https://hexo.io/zh-cn/docs/ 查看更多的配置。

## 部署脚本
```
hexo clean
hexo generate
hexo deploy
```
