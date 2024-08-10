---
title: "基于 Hugo 搭建自己的博客"
date: 2022-11-04T18:22:26+08:00
lastmod: 2022-11-04T18:22:26+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["geek", "hugo", "tips"]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---

## 前言 

Hugo 是用 Go 语言写的静态网站生成器（Static Site Generator）。可以把 Markdown 文件转化成 HTML 文件，因此有很多人利用 Hugo 来搭建自己的博客网站。

## 安装 Hugo

Mac 上，执行 `brew install hugo`，Win 上执行 `scoop install hugo` 即可。使用 `hugo new site test-coder` 即可创建博客，该命令会在当前目录创建一个名为 `test-coder` 的子目录，该目录就是创建的博客的源文件仓库。其中，`test-coder` 可以自行修改为你希望的名称。该目录中的内容如下：

```txt
test-coder on  main [+?] 
❯ ls                      
archetypes assets     content    data       hugo.toml  i18n       layouts    public     resources  static     themes
```

- `archetypes`：存放 front matter 模板，`hugo` 命令创建 `.md` 文件时会根据该模板来创建；
- `content`：存放博客内容；
- `layouts`：存放定义为网站的样式，写在 `layouts` 目录下的样式文件会覆盖安装的主题中的 `layouts` 目录下的同名样式文件；
- `static`：存放静态文件，`static` 目录中的内容会在编译时会被移动到 `public` 目录，而本地的 `public` 目录对应着网站的根目录；
- `public`：存放 hugo 生成的静态网页；
- `themes`：存放主题文件；
- `config.toml`：网站配置文件，也可能是 `hugo.toml`。

## 安装 Hugo 主题

这里我选择的是简洁纯粹的 [hugo-coder](https://github.com/luizdepra/hugo-coder/) 主题。

在 `test-coder` 目录下执行 `git submodule add https://github.com/luizdepra/hugo-coder.git themes/coder` 即可安装主题。

安装好主题之后，则需要根据自己的需求修改 `hugo.toml`，可以参考 hugo-coder 作者给出的 [示例](https://github.com/luizdepra/hugo-coder/blob/main/exampleSite/hugo.toml)，注意 `baseURL` 处要修改成自己的博客网址。

```toml
baseURL = "http://blog.zwyyy456.tech" # 改为自己的博客域名
title = "zwyyy456's blog"
theme = "coder" # 注意你选择的主题名
languageCode = "en"
defaultContentLanguage = "en"
paginate = 6
enableEmoji = true

[services]
[services.disqus]

[markup.highlight]
noClasses = false

[params]
author = "zwyyy456"
# license = '<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">CC BY-SA-4.0</a>'
description = "笔记与杂谈"
keywords = "tech, blog, life"
info = ["IT", "Running"]
avatarURL = "images/Quirrel.jpg"
#gravatar = "john.doe@example.com"
dateFormat = "January 2, 2006"
since = 2022
# Git Commit in Footer, uncomment the line below to enable it
commit = "https://github.com/luizdepra/hugo-coder/tree/"
# Right To Left, shift content direction for languages such as Arabic
rtl = false
colorScheme = "auto"
# Hide the toggle button, along with the associated vertical divider
hideColorSchemeToggle = false
# Series see also post count
maxSeeAlsoItems = 5
# Custom CSS
customCSS = []
# Custom SCSS, file path is relative to Hugo's asset folder (default: {your project root}/assets)
customSCSS = []
# Custom JS
customJS = []
# Custom remote JS files
customRemoteJS = []

[taxonomies]
category = "categories"
series = "series"
tag = "tags"
author = "authors"

[[params.social]]
name = "Github"
icon = "fa-brands fa-github fa-2x"
weight = 1
url = "https://github.com/zwyyy456/"

[[params.social]]
name = "Twitter"
icon = "fa-brands fa-x-twitter fa-2x"
weight = 3
url = "https://twitter.com/zwyyy456/"

[[params.social]]
name = "LinkedIn"
icon = "fa-brands fa-linkedin fa-2x"
weight = 4
url = "https://www.linkedin.com/in/zwyyy456/"

[[params.social]]
name = "RSS"
icon = "fa-solid fa-rss fa-2x"
weight = 6
url = "https://myhugosite.com/index.xml"
rel = "alternate"
type = "application/rss+xml"

[languages.en]
languageName = ":us:"

[[languages.en.menu.main]]
name = "Search"
weight = 1
url = "search/"

[[languages.en.menu.main]]
name = "About"
weight = 2
url = "about/"

[[languages.en.menu.main]]
name = "Blog"
weight = 3
url = "posts/"

[[languages.en.menu.main]]
name = "Tech"
weight = 4
url = "posts/tech"

[[languages.en.menu.main]]
name = "Fun"
weight = 5
url = "posts/blog"


[languages.zh]
languageName = ":cn:"
title = "翼仔的博客"

[languages.zh.params]
author = "翼仔"
info = "何以解忧"
description = "唯有杜康"
keywords = "tech, blog, life"

[[languages.zh.menu.main]]
name = "搜索"
weight = 1
url = "search/"

[[languages.zh.menu.main]]
name = "关于"
weight = 2
url = "about/"

[[languages.zh.menu.main]]
name = "文章"
weight = 3
url = "posts/blog"

[[languages.zh.menu.main]]
name = "技术"
weight = 4
url = "posts/tech"

[[languages.zh.menu.main]]
name = "折腾"
weight = 5
url = "posts/blog"
```

通过 `hugo new posts/tech/xxx.md` 会在 `content/posts/tech` 目录下根据 `archetypes` 中的 `front matter` 模板生成 `xxx.md` 文件，默认语言是 en，该博客对应的网址是 `https://blog.zwyyy456.tech/posts/tech/xxx/`，如果执行 `hugo new posts/tech/xxx.zh.md`，则会生成 zh-cn 版的博客，对应网址为 `https://blog.zwyyy456.tech/zh/posts/tech/xxx`，点击如下图的旗帜图标，可以切换博客的语言版本（假如有的话）。

![Qzf19G](https://pic-upyun.zwyyy456.tech/uPic/Qzf19G.png)

对 PaperMod 主题，是由 `archetypes` 中的 `default.md` 作为 `font matter` 模板，然而 hugo-coder 主题是 `posts.md`。模板内容如下：

```yaml
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
lastmod: {{ .Date }} #更新时间
author: ["zwyyy456"] #作者
categories: [""]
tags: [""]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---
```

> 注意这里 `comments` 暂时不能设置为 `true`，否则会编译失败，留待后续解决，毕竟我也不需要评论系统。

## 目录设置

在 `content/posts` 目录下，我创建了 `tech`、`blog`、`read` 这四个目录，对于 `papermod` 主题，以 `tech` 目录为例，我们需要在对应目录下创建 `_index.md` 和 `_index.zh.md`，才能访问到 `https://blog.zwyyy456.tech/zh/posts/tech` 和 `https://blog.zwyyy456.tech/posts/tech`，否则对应地址会显示为 404。`index.zh.md` 的内容如下：

```yaml
--- 
title: "技术"
description: "种一棵树最好的时间是十年前，其次是现在"
hidemeta: true
---
```

而 `hugo-coder` 主题则不需要创建 `_index.md`。

## 部署博客到vercel

### 购买域名

我的域名 `zwyyy456.tech` 是在阿里云处购买的，同时也是托管在阿里云上的，价格为 10 年 188 元，根据自己的需求购买即可。我的域名解析也是在阿里云上的。

### Netlify 部署博客

将整个项目，如 `test-coder` 这个文件夹，作为一个 git 仓库上传到 GitHub，然后 Netlify 点击 `Add new site` -> `Import an existing project`，选择 `Deploy with GitHub`，就能将对应的仓库导入到 Netlify，部署的时候设置一下站点名，其他默认即可。

![RVDWQv](https://pic-upyun.zwyyy456.tech/uPic/RVDWQv.png)

项目部署好之后，添加自己的域名，以 `test-blog.zwyyy456.tech` 为例，添加为 subdomain，然后点击 `External DNS`，根据给出的 CNAME 信息去阿里云添加域名解析，如下图所示：

![AictqH](https://pic-upyun.zwyyy456.tech/uPic/AictqH.png)

![P101c2](https://pic-upyun.zwyyy456.tech/uPic/P101c2.png)

解析生效之后就能通过 `test-blog.zwyyy456.tech` 域名访问博客了，之后，每次 `git push` 到 GitHub，Netlify 都会重新部署网站以使内容保持最新。

## 添加图床以及网站备案

我使用的图床是白嫖的又拍云，而使用又拍云国内图床，需要一个备案过的域名，因此就需要对前面申请的域名进行备案。考虑到域名是通过阿里云购买的，因此也通过阿里云进行备案，于是，我们需要一个阿里云的备案服务号，而备案服务号则需要你有阿里云的服务器（至少一年），当然，闲鱼似乎也有卖备案服务号。

由于之前备案的时候没有做记录，这里就不再演示，跟着阿里云的备案流程走即可，注意备案时网站名称不要出现博客或者论坛等字样，建议“项目笔记”。

又拍云图床的使用教程参见 [将博客的图床从 SM.MS 迁移到又拍云 ](https://blog.zwyyy456.tech/zh/posts/blog/change-image-src-from-smms-to-upyun/)。

由于免费使用又拍云图床以及备案，都需要在网站底部显示对应内容，因此在 `layouts/partials` 目录下创建 `footer.html`，内容如下：

```html
<footer class="footer">
    <section class="container">
        ©
        {{ if (and .Site.Params.since (lt .Site.Params.since now.Year)) }}
        {{ .Site.Params.since }} -
        {{ end }}
        {{ now.Year }}
        {{ with .Site.Params.author }} {{ . }} {{ end }}
        ·
        {{ if (and .Site.Params.license) }}
        {{ i18n "licensed_under" }} {{ .Site.Params.license | safeHTML }}
        ·
        {{ end }}
        {{ i18n "powered_by" }} <a href="https://gohugo.io/" target="_blank" rel="noopener">Hugo</a> & <a
            href="https://github.com/luizdepra/hugo-coder/" target="_blank" rel="noopener">Coder</a> & <a
            href="https://www.upyun.com/?utm_source=lianmeng&utm_medium=referral">
            <img src='/images/upyun.svg' alt="又拍云" width="53" height="18"
                style="fill: currentColor; position: relative; top: +3.5px;">
        </a>.
        {{ if (and .Site.Params.commit .GitInfo) }}
        [<a href="{{ .Site.Params.commit }}/{{ .GitInfo.Hash }}" target="_blank" rel="noopener">{{
            .GitInfo.AbbreviatedHash }}</a>]
        {{ end }}
        <br>
        <a href="https://beian.miit.gov.cn/" target="_blank">湘ICP备2023038416号</a>
    </section>
</footer>
```

最后显示效果如下：

![Q07Mw8](https://pic-upyun.zwyyy456.tech/uPic/Q07Mw8.png)

为博客添加内容搜索功能参见 [使博客被搜索引擎收录](https://blog.zwyyy456.tech/zh/posts/blog/blog-google-bing)。

## 参考

[ 如何用 GitHub Pages + Hugo 搭建个人博客 ](https://cuttontail.blog/blog/create-a-wesite-using-github-pages-and-hugo/)
[hugo博客搭建 | PaperMod主题 ](https://www.sulvblog.cn/posts/blog/build_hugo/)