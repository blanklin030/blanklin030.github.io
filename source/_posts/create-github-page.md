---
title: 如何创建GitHub Page
date: 2019-10-29 11:47:21
tags:
  - hexo
categories:
  - tools
---

## 新建项目

首先在 GitHub 新建一个仓库（Repository），名称为 {username}.github.io，注意这个名比较特殊，必须要是 github.io 为后缀结尾的。比如 我 的 GitHub 用户名就叫 blanklin030，那我就新建一个 blanklin030.github.io，新建完成之后就可以进行后续操作了。

> > 另外如果 GitHub 没有配置 SSH 连接的建议配置一下，这样后面在部署博客的时候会更方便。

## 安装环境

首先在自己的电脑上安装 Node.js，下载地址：https://nodejs.org/zh-cn/download/，可以安装 Stable 版本。

安装完毕之后，确保环境变量配置好，能正常使用 npm 命令。

## 安装 Hexo

接下来就需要安装 Hexo 了，这是一个博客框架，Hexo 官方还提供了一个命令行工具，用于快速创建项目、页面、编译、部署 Hexo 博客，所以在这之前我们需要先安装 Hexo 的命令行工具。

命令如下：

```
npm install -g hexo-cli
```

安装完毕之后，确保环境变量配置好，能正常使用 hexo 命令。

## 初始化项目

接下来我们使用 Hexo 的命令行创建一个项目，并将其在本地跑起来，整体跑通看看。
首先使用如下命令创建项目：

```
hexo init {name}
```

这里的 name 就是项目名，我这里要创建 blanklin 的博客，我就把项目取名为 blanklin 了，用了纯小写，命令如下：

```
hexo init blinklin
```

这样 nightteam 文件夹下就会出现 Hexo 的初始化文件，包括 themes、scaffolds、source 等文件夹，这些内容暂且先不用管是做什么的，我们先知道有什么，然后一步步走下去看看都发生了什么变化。

接下来我们首先进入新生成的文件夹里面，然后调用 Hexo 的 generate 命令，将 Hexo 编译生成 HTML 代码，命令如下：

```
hexo generate
```

可以看到输出结果里面包含了 js、css、font 等内容，并发现他们都处在了项目根目录下的 public 文件夹下面了。

然后我们利用 Hexo 提供的 serve 命令把博客在本地运行起来，命令如下：

```
hexo serve
```

运行之后命令行输出如下：

```
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

## 部署

接下来我们来将这个初始化的博客进行一下部署，放到 GitHub Pages 上面验证一下其可用性。成功之后我们可以再进行后续的修改，比如修改主题、修改页面配置等等。

那么怎么把这个页面部署到 GitHub Pages 上面呢，其实 Hexo 已经给我们提供一个命令，利用它我们可以直接将博客一键部署，不需要手动去配置服务器或进行其他的各项配置。

部署命令如下：

```
hexo deploy
```

在部署之前，我们需要先知道博客的部署地址，它需要对应 GitHub 的一个 Repository 的地址，这个信息需要我们来配置一下。

打开根目录下的 \_config.yml 文件，找到 Deployment 这个地方，把刚才新建的 Repository 的地址贴过来，然后指定分支为 master 分支，最终修改为如下内容：

```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: {git repo ssh address}
  branch: master
```

另外我们还需要额外安装一个支持 Git 的部署插件，名字叫做 hexo-deployer-git，有了它我们才可以顺利将其部署到 GitHub 上面

```
npm install hexo-deployer-git --save
```

安装成功之后，执行部署命令：

```
hexo deploy
```

如果没有报错，那说明 hexo deploy 操作就做了两个动作，1 个是 build 文件到 public 目录，还有 1 个就是 git add && git commit && git push 后到 GitHub 仓库了。
这时候我们去 GitHub 上面看看 Hexo 上传了什么内容，打开之后可以看到 master 分支就可以看到其实就是把 public 目录 push 到 repository 了。

这时候可能就有人有疑问了，那我博客的源码也想放到 GitHub 上面怎么办呢？其实很简单，新建一个其他的分支就好了，比如我这边就新建了一个 source 分支，代表博客源码的意思。

具体的添加过程就很简单了，就是增加一个 source 分支，记录的博客的源码，参加如下命令：

```

git init
git checkout -b source
git add -A
git commit -m "init blog"
git remote add origin git@github.com:{username}/{username}.github.io.git
git push origin source
```

成功之后，可以到 GitHub 上再切换下默认分支，比如我就把默认的分支设置为了 source，当然不换也可以。

## 配置站点信息

下面我就以自己的站点 NightTeam 为例，修改一些基本的配置，比如站点名、站点描述等等。

修改根目录下的 \_config.yml 文件，找到 Site 区域，这里面可以配置站点标题 title、副标题 subtitle 等内容、关键字 keywords 等内容，比如我的就修改为如下内容：

```
# Site
title: NightTeam
subtitle: 一个专注技术的组织
description: 涉猎的主要编程语言为 Python、Rust、C++、Go，领域涵盖爬虫、深度学习、服务研发和对象存储等。
keywords: "Python, Rust, C++, Go, 爬虫, 深度学习, 服务研发, 对象存储"
author: NightTeam
```

这里大家可以参照格式把内容改成自己的。

另外还可以设置一下语言，如果要设置为汉语的话可以将 language 的字段设置为 zh-CN，修改如下：

```
language: zh-CN
```

### 修改主题

目前 Hexo 里面应用最多的主题基本就是 Next 主题了，个人感觉这个主题还是挺好看的，另外它支持的插件和功能也极为丰富，配置了这个主题，我们的博客可以支持更多的扩展功能，比如阅览进度条、中英文空格排版、图片懒加载等等。

那么首先就让我们来安装下 Next 这个主题吧，目前 Next 主题已经更新到 7.x 版本了，我们可以直接到 Next 主题的 GitHub Repository 上把这个主题下载下来。

主题的 GitHub 地址是：https://github.com/theme-next/hexo-theme-next，我们可以直接把 master 分支 Clone 下来。

首先命令行进入到项目的根目录，执行如下命令即可：

```
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

执行完毕之后 Next 主题的源码就会出现在项目的 themes/next 文件夹下。

然后我们需要修改下博客所用的主题名称，修改项目根目录下的 \_config.yml 文件，找到 theme 字段，修改为 next 即可，修改如下：

```
theme: next
```

然后本地重新开启服务，访问刷新下页面，就可以看到 next 主题就切换成功了

### 主题配置

现在我们已经成功切换到 next 主题上面了，接下来我们就对主题进行进一步地详细配置吧，比如修改样式、增加其他各项功能的支持，下面逐项道来。

Next 主题内部也提供了一个配置文件，名字同样叫做 \_config.yml，只不过位置不一样，它在 themes/next 文件夹下，Next 主题里面所有的功能都可以通过这个配置文件来控制，下文所述的内容都是修改的 themes/next/\_config.yml 文件。

#### 样式

Next 主题还提供了多种样式，风格都是类似黑白的搭配，但整个布局位置不太一样，通过修改配置文件的 scheme 字段即可，我选了 Pisces 样式，修改 \_config.yml （注意是 themes/next/\_config.yml 文件）如下：

```
scheme: Pisces
# 大家可以自行根据喜好选择。
# scheme: Muse
#scheme: Mist
#scheme: Gemini
```

> 还有很多其他的配置项，可自己参考修改，在这里就不一一概述了

最后大家修改完配置后请[参考文档](https://blanklin030.github.io/2019/10/29/write-article/)继续学习如何创建文章。
