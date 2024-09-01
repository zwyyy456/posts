---
title: "将博客的图床从 SM.MS 迁移到又拍云"
date: 2023-12-26T16:25:29+08:00
lastmod: 2023-12-26T16:25:29+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["blog", "hugo", "image"]
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
为了存储博客的图片，我需要一个图床，一开始本来是想用七牛云或者又拍云提供的免费存储服务和流量来做图床的，奈何它们都需要一个通过了备案的域名才能实现，于是退而求其次选择了 SM.MS，在 picgo 或者 upic 中配置 SM.MS 图床非常方便，填一个 token 即可，在科学加持的情况下，图片显示也是很快速的。

但是，由于我的博客很多内容是我自己的学习笔记，因此其中的图片内容也是很重要的，考虑到 SM.MS 的图床稳定性与国内的可直接访问性，当我的 `zwyyy456.tech` 域名通过备案之后，我就着手将图床从 SM.MS 迁移到了又拍云。

## 又拍云设置

又拍云联盟的用户每月可获得 10GB 免费存储空间与 15GB 免费 CDN 流量，要加入又拍云联盟也很简单，只需要在网站地步添加又拍云 LOGO 与又拍云网站的对应链接即可。

在博客所属根目录下，将 `layouts/partials/foot.html` 的内容修改为如下：

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
            <img src='/img/upyun.svg' alt="又拍云" width="53" height="18"
                style="fill: currentColor; position: relative; top: +3.5px;">
        </a>.
        {{ if (and .Site.Params.commit .GitInfo) }}
        [<a href="{{ .Site.Params.commit }}/{{ .GitInfo.Hash }}" target="_blank" rel="noopener">{{
            .GitInfo.AbbreviatedHash }}</a>]
        {{ end }}
        <br>
        <a href="https://beian.miit.gov.cn/" target="_blank">湘 ICP 备 2023038416 号</a>
    </section>
</footer>
```

顺便满足了备案的要求。

完成备案之后，进入又拍云控制台，“云存储服务”那里点击 `创建服务`，服务名称命名为 `image-zwyyy`（不可以和全网已有的服务名称重复），`应用场景`、`存储类型`、`加速区域` 都选择默认，`授权操作员` 选择 `新建授权操作员`，`操作员` 与 `密码` 这两栏自己定义，权限全部勾选，如下图：
 
![OhTenv](https://pic-upyun.zwyyy456.tech/uPic/OhTenv.jpg)

创建好之后，进入 `配置`，在 `域名管理` 的这一栏下面，点击 `域名绑定`，绑定已备案域名的子域名，例如 `test-upyun.zwyyy456.tech`，输入该网址之后，要点击 Enter 才能点击 `添加`，如下图所示：

![ryO0G8](https://pic-upyun.zwyyy456.tech/uPic/ryO0G8.png)

在阿里云处添加对应的 CNAME 解析，如下图所示：

![Gj0brD](https://pic-upyun.zwyyy456.tech/uPic/Gj0brD.png)

然后进入 `HTTPS` 这一栏，点击 `HTTPS 配置` 旁边的管理，去 `证书管理` 处申购免费证书：

![r6M7iU](https://pic-upyun.zwyyy456.tech/uPic/r6M7iU.png)

![MdKetn](https://pic-upyun.zwyyy456.tech/uPic/MdKetn.png)

在 `证书申购` 处点击 `补全` 以补充相关信息：

![mwijUV](https://pic-upyun.zwyyy456.tech/uPic/mwijUV.png)

之后点击提交，在 `申请结果` 处点击 `操作` 一栏下的 `查看`，在阿里云处添加对应的域名解析：

![G7UYxz](https://pic-upyun.zwyyy456.tech/uPic/G7UYxz.png)

添加好之后结果如下：

![19HZuX](https://pic-upyun.zwyyy456.tech/uPic/19HZuX.png)

之后再在又拍云处点击验证，结果如下：

![VZoIb6](https://pic-upyun.zwyyy456.tech/uPic/VZoIb6.png)

然后即可回到 `存储服务` 的 `HTTPS` 配置处以开启 HTTPS 访问和强制 HTTPS 访问。

## 通过 iPic Mover 迁移图床

之前尝试通过 picgo 的 migrator 插件迁移图床，发现它无法下载存储在 SM.MS 图床上的照片，后来发现了 iPic Mover  这个工具，它依托于 iPic，iPic 配置好图床之后，即可通过 iPic Mover 进行迁移，非常方便，不过 iPic 需要付费才能自定义图床，但是可以试用七天，完成迁移之后卸载掉即可。

此外，网上也有一些迁移的教程，大概是利用 SM.MS 的 API，写个代码，请求下载对应的图片，并上传至又拍云，然后将链接替换为又拍云的链接。

## Vercel + CloudFlare 重定向次数过多的问题

进入 CloudFlare Dashboard，点击有问题的域名，打开左侧的 SSL/TLS 设置，在 Overview 中设置加密模式为完全或完全（严格）即可。

![Sy3mpw](https://pic-upyun.zwyyy456.tech/uPic/Sy3mpw.png)

## 解决“代理情况下无法加载存放于又拍云对象存储的图片”的问题

在 `rule-provider` 目录下 的 `direct-rule.yaml` 中，追加两条规则：

```yaml
  - DOMAIN-SUFFIX,b0.aicdn.com

  # upyun
  - DOMAIN-SUFFIX,pic-upyun.zwyyy456.tech
```