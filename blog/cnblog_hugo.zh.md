---
title: "便捷同步本地的博客文档到博客园"
date: 2023-06-14T14:22:30+08:00
lastmod: 2023-06-14T14:22:30+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["trick"]
description: "" #描述
weight: # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---
## 前言

我写博客的初心很简单，一是一些软件的配置过程（防止第二次配置的时候又抓瞎）；二是记录下一下自己学习过程中的一些心得体会，在 [高乙超的博客](https://gaoyichao.com/Xiaotu/?book=diary&title=about) 中，我曾经看到一句话，叫 **"To learn, read; To know, write; To master, teach"**。

过去二十年里，在学习的过程中，一直是作为一个输入方，应付考试倒是没啥问题，但也仅此而已了。为了更好地体会和领悟这些知识，我决定写写博客，既是做笔记，也是对自己的所学的一种整理和输出，也希望能有更多同道者看到，从而一起交流学习、共同进步。

最一开始么，其实是告诫自己，写博客的核心在于动笔输出，而不是折腾博客的主题又或是如何搭建博客站点，因此就想着选择一个现有的公开的博客平台，经过一番比较之下选择了博客园。说是比较，其实也没得太多选择的余地，除了博客园也就是 CSDN 了，然而对于 CSDN 我实在是深恶痛绝。博客园相较之下克制很多，广告少，更聚焦于技术，原创内容更多且更有深度，虽然界面以 2023 年的眼光来看比较老土了，但博客园支持自定义 CSS 啊。

没错，我还是无法控制自己，折腾了一下博客园的主题，鉴于本人对前端一窍不通，就算以后了解了，以我这种纠结来纠结去的性子，自己动手写主题，选择配色、主题、字号等绝对是噩梦，因此仅限于在 GitHub 上搜索他人做好的主题，挑了一番之后，网上比较热门的诸如 Silence 之类的主题，我都觉得太花里胡哨了，而且字体并不喜欢，找到一个设计风格不错的，奈何主题又太久没更新了，最后作罢。

最后，我还是选择了博客园自带的 **Coding Life** 主题，修改了一下代码块的 CSS，主要是改了代码块字体。

然而写了几篇博客之后，又发现博客园自带的编辑器太难用了。还是要自己现在本地用 VsCode 配合 Markdown Preview Enhanced 插件写好，再复制粘贴到博客园上发布，就觉得有点麻烦，倒不如用 Hugo 或者 Hexo 了，配置好之后，写好博客再敲一下命令就能搞定了。

于是我又动了用 Hexo 或者 Hugo 自己搭一个博客网站的念头，首先简单对比了一下，选择了 Hugo，比起 Hexo 来说，它性能高，配置起来更方便，Hexo 非常流行的 Next 主题我也并不喜欢，倒是 Hugo 的不少主题我非常喜欢，我个人在用的主题是 [hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod) 和 [hugo-coder](https://github.com/luizdepra/hugo-coder)，没错，我一口气搭了两个网站，在我看来 **hugo-coder** 更好看一点，而 **hugo-PaperMod** 功能更为齐全，用的人更多，教程也更多。

两个博客网址分别是 [zwyyy456.tech](https://blog.zwyyy456.tech/) 和 [paper.zwyyy456.tech](https://paper.zwyyy456.tech)。

我这两个基于 Hugo 的博客可以说是纯白嫖实现的，域名是白嫖的 Freenom 的 `.tk` 和 `.ml` 域名，自动部署方案是基于 GitHub 和 Vercel 实现的，将本地修改同步到 GitHub 之后，Vercel 就会自动构建站点。

这个方案我用了快半年，总体上来说是很满意的，写起博客来非常方便，站点颜值也很高，唯一美中不足的是，Google 或者 百度基本没有人看，估计和用的白嫖域名有关吧，我也不懂 SEO 优化，也没时间折腾这些，还有就是没有评论系统（反正也没人看）。其实搭建好个人博客之后，我还是坚持用复制粘贴的方式把博客同步到博客园坚持了一段时间的，后来太懒了，作罢。

就在前两天，我在查找 C++ 的内存分布相关资料的时候，发现了博主 sinkinben 的这篇 [文章](https://www.cnblogs.com/sinkinben/p/15475940.html)，我承认，肤浅的我一下子被他的博客主题吸引到了（内容反倒是其次了），在他的博客中有给出他的 Github 链接，点进去一看，果然有他的博客园主题的 CSS 和 html 文件，更改了一下博客园主题，看了一下实际效果，果然很好看。于是，我那把文章同步发布在博客园的心又蠢蠢欲动了起来。

## 博客园同步脚本

其实我一直有在关注博客园的 VsCode 插件，也使用过，其实挺好用的，但是有一个非常致命的问题。那就是执行 `hugo new xxx.md` 生成的文档都会带有如下图的 Yaml Front Matter 信息，这些信息不属于正文内容，Hugo 会基于这些信息解析出本文的标题、所属分类、标签等信息。而插件在上传文章到博客园时，并不会解析或者去除这些信息，而是作为正文的一部分直接上传，GitHub 上已经有人提 issue 了，开发者也注意到了这个问题，但还没来得及修改。

因此，如果想便捷地同步文章到博客园，在插件开发者做出修改之前，我只能另寻出路。经过我的一番检索，我发现了一个基本符合我需求的发布文章到博客园的 Python 脚本，基础教程见该脚本的 [GitHub 地址](https://github.com/DeppWang/cnblogs-post)。

首先，需要执行 `conda install markdown-it-py` 和 `conda install mdit-py-plugins` 安装所需要的模块，

配置好脚本文件中对应的文章路径、MetaWeblog 账户信息之后，运行该脚本即可，运行时后面可以带一个数字作为参数，表示一次更新或者发布多少文章，脚本会自动发布指定数量的最近修改过的文章到博客园，如果未指定则参数默认为 $1$。

注意，运行脚本时，如果碰到 Yaml Front Matter 部分的 tags 一栏为空的 `.md` 文件，脚本会直接结束运行，同时 Hugo 限制了 `_index.zh.md` 不能有 tags 这一栏，因此要特别作一点处理，但其实只要保证最近更新的文章不包括 `_index.zh.md` 即可。

该脚本只有原来只能处理 tags，对于 categories 这一栏则不会处理，因此我做了一点修改。这里要注意 MetaWeblog API 的 `categories` 字段应该是一个字符串列表，作者对 tags 是作为字符串来处理的。

```py
import xmlrpc.client
import ssl
import os
import sys
import logging

# logging.basicConfig(level=logging.INFO)

# 设置 ssl 很重要，否则将报 ssl 错
ssl._create_default_https_context = ssl._create_unverified_context

# dict
config = {
    "user_unique_name": "zwyyy456",  # 你的用户名，用于拼接文章 url
    "url": "https://rpc.cnblogs.com/metaweblog/zwyyy456",  # 你的 MetaWeblog 访问地址
    "username": "zwyyy456",  # 你的登录用户名，可能跟上面的不一致
    "password": "*********",  # 你的登录密码
    "local_post_path": "/home/zwyyy/Documents/.../",  # 你的本地博文路径
}


class MetaWeblog:
    def __init__(self, url, username, password):
        self.url, self.username, self.password = url, username, password
        self.proxy = xmlrpc.client.ServerProxy(self.url)

    def getRecentPosts(self, count):
        return self.proxy.metaWeblog.getRecentPosts(
            "", self.username, self.password, count
        )

    def deletePost(self, post_id):
        return self.proxy.blogger.deletePost(
            "", post_id, self.username, self.password, True
        )

    def newPost(self, article):
        return self.proxy.metaWeblog.newPost(
            "",
            self.username,
            self.password,
            dict(
                title=article["title"],
                description=article["content"],
                categories=article["categories"],
                mt_keywords=article["tags"],
            ),
            True,
        )

    def editPost(self, post_id, article):
        return self.proxy.metaWeblog.editPost(
            post_id,
            self.username,
            self.password,
            dict(
                title=article["title"],
                description=article["content"],
                mt_keywords=article["tags"],
                categories=article["categories"],
            ),
            True,
        )


def set_article(article_path: str) -> dict:
    """根据文章路径设置文章（标题、标签和内容）"""

    import re
    from markdown_it import MarkdownIt
    from mdit_py_plugins.front_matter import front_matter_plugin
    from mdit_py_plugins.footnote import footnote_plugin

    with open(article_path, "rb") as f:
        content = f.read().decode("utf-8")

    # 使用分组
    reg = r"(---(.|[\n])*?---)"

    p = re.compile(reg)
    headers = p.findall(content)

    logging.info("headers: %s", headers)
    if len(headers) < 1:
        raise ValueError("文章 %s 不存在 header 信息块「--- ** ---」，请检查！" % article_path)

    # headers[0] 为 tuple
    header_str = headers[0][0]

    lines = header_str.split("\n")

    title, tags_categories, category = "", [], ""
    for line in lines:
        # 不使用':', 再替换所有空格为 null，因为可能存在中英文空格
        line_ele = line.split(": ")
        if line_ele[0] == "title":
            title = line_ele[1].rstrip()
            title = title.strip('"')  # 去除首尾的引号

        elif line_ele[0] == "tags":
            tags = line_ele[1].rstrip()
            tags = tags.replace("[", "")
            tags = tags.replace('"', "")
            tags = tags.replace("]", "")
            tags_categories.append(tags)

        # 处理 categories
        elif line_ele[0] == "categories":
            category = line_ele[1].rstrip()
            category = category.replace("[", "")
            category = category.replace('"', "")
            category = category.replace("]", "")

    separator = ", "
    tags = separator.join(tags_categories)

    categories = category.split(", ")

    logging.info("tags: %s", tags)
    logging.info("categories: %s", categories)

    # 只替换第一个
    content = content.replace(header_str, "", 1)

    md = (
        MarkdownIt()
        .use(front_matter_plugin)
        .use(footnote_plugin)
        .enable("image")
        .enable("table")
    )

    # markdown 转换为 HTML
    content = md.render(content)

    # cnblogs-markdown 属性用于代码块样式
    content = '<div class="cnblogs-markdown">%s</div>' % content

    if title == "" or content == "" or tags == "":
        raise ValueError("文章 title、content、tags 均不能为空；': '后有空格。请检查！")

    article = {
        "title": title,
        "content": content,
        "tags": tags,
        "categories": categories,
    }
    return article


def get_local_modified_file(file_count: int):
    """获取指定数量的最近修改文章"""

    import stat
    import datetime as dt

    modified = []

    for root, sub_folders, files in os.walk(config["local_post_path"]):
        for file in files:
            try:
                unix_modified_time = os.stat(os.path.join(root, file))[stat.ST_MTIME]
                human_modified_time = dt.datetime.fromtimestamp(
                    unix_modified_time
                ).strftime("%Y%m%dT%H:%M:%S")
                filename = os.path.join(root, file)
                if os.path.splitext(filename)[1] == ".md":
                    modified.append((human_modified_time, filename))
            except:
                pass

    modified.sort(key=lambda a: a[0], reverse=True)
    return modified[0:file_count]


def judge_str_equal(str1, str2):
    """判断两个字符串是否相等"""

    str1 = str1.replace(" ", "")
    str2 = str2.replace(" ", "")
    return str1 == str2


def edit_or_new(article_path: str) -> None:
    """编辑或新增"""

    blog = MetaWeblog(config["url"], config["username"], config["password"])
    posts = blog.getRecentPosts(100)
    article = set_article(article_path)

    title = article["title"]

    # 如果存在，更新，否则新增。接口不提供最近修改时间，所以不能跳过
    exist_flag = 0

    for post in posts:
        # 判断是否含有相同的字符，不直接用等号
        if judge_str_equal(post["title"], title):
            status = blog.editPost(post["postid"], article)
            if status is True:
                print(
                    "更新文章「%s」成功，文章地址 https://www.cnblogs.com/%s/p/%s.html"
                    % (title, config["user_unique_name"], post["postid"])
                )
            exist_flag = 1
            break

    if exist_flag == 0:
        id = blog.newPost(article)
        if id is not None:
            print(
                "发布文章「%s」成功，文章地址 https://www.cnblogs.com/%s/p/%s.html"
                % (title, config["user_unique_name"], id)
            )


def post(count: int) -> None:
    """发布指定数量的最新修改文章"""

    modified_files = get_local_modified_file(count)
    for modified_file in modified_files:
        edit_or_new(modified_file[1])


def delete() -> None:
    """默认删除最新文章"""

    metaWeblog = MetaWeblog(config["url"], config["username"], config["password"])
    posts = metaWeblog.getRecentPosts(100)
    postid = posts[0]["postid"]
    title = posts[0]["title"]
    status = metaWeblog.deletePost(postid)
    if status is True:
        print("删除「%s」成功！" % title)
    else:
        print("删除「%s」失败，文章不存在。文章 id 为：%s" % (title, postid))


def main():
    try:
        if len(sys.argv) > 1 and sys.argv[1] == "delete":
            delete()

        elif len(sys.argv) > 1 and isinstance(int(sys.argv[1]), int):
            print("正在发布 ...")
            logging.info("sys.argv[1]: %s", sys.argv[1])
            post(int(sys.argv[1]))

        else:
            post(1)
    except Exception as err:
        print("错误提示：%s", format(err))
        print("发布失败")
        sys.exit(-1)


if __name__ == "__main__":
    main()

```

另外还有一个执行三个不同的 Python 脚本的脚本。

```sh
python3 ./CnblogPy/tech_posts.py 6
python3 ./CnblogPy/leet_posts.py 6
python3 ./CnblogPy/blog_posts.py 6
```

## 修改主题

博主 sinkinben 制作的主题已经非常完善了，我主要是针对自己需求修改了一下 `header.html`。

一是将 🌳 Home、📮 Email、🐶 About 这三个 `menuItem` 对应的链接修改为自己的相关链接，而是添加了 🏷️ Tag 和 📂 Category 以及相关的 `subMenuItem` 链接，并将头像换成了自己的。



