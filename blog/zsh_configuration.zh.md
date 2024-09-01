---
title: "Zsh 的简单配置与使用"
date: 2023-03-23T10:55:08+08:00
lastmod: 2023-03-23T10:55:08+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["zsh", "tips"]
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
## 安装 starship
MacOS 下执行`curl -sS https://starship.rs/install.sh | sh`即可，Linux 下（如 Debian），执行`curl -sS https://starship.rs/install.sh | sh`即可，然后执行`starship preset pure-preset > ~/.config/starship.toml`将预设主题设置为**pure**。

## 安装 zinit 插件
执行`bash -c "$(curl --fail --show-error --silent --location https://raw.githubusercontent.com/zdharma-continuum/zinit/HEAD/scripts/install.sh)"`以自动安装**zinit。

修改`~/.zshrc`为以下内容
```sh
## 启用starship
eval "$(starship init zsh)"


### Added by Zinit's installer
if [[ ! -f $HOME/.local/share/zinit/zinit.git/zinit.zsh ]]; then
    print -P "%F{33} %F{220}Installing %F{33}ZDHARMA-CONTINUUM%F{220} Initiative Plugin Manager (%F{33}zdharma-continuum/zinit%F{220})…%f"
    command mkdir -p "$HOME/.local/share/zinit" && command chmod g-rwX "$HOME/.local/share/zinit"
    command git clone https://github.com/zdharma-continuum/zinit "$HOME/.local/share/zinit/zinit.git" && \
        print -P "%F{33} %F{34}Installation successful.%f%b" || \
        print -P "%F{160} The clone has failed.%f%b"
fi

source "$HOME/.local/share/zinit/zinit.git/zinit.zsh"
autoload -Uz _zinit
(( ${+_comps} )) && _comps[zinit]=_zinit

# Load a few important annexes, without Turbo
# (this is currently required for annexes)
zinit light-mode for \
    zdharma-continuum/zinit-annex-as-monitor \
    zdharma-continuum/zinit-annex-bin-gem-node \
    zdharma-continuum/zinit-annex-patch-dl \
    zdharma-continuum/zinit-annex-rust

### End of Zinit's installer chunk

# 快速目录跳转
zinit ice lucid wait='1'
zinit light skywind3000/z.lua

# 语法高亮
zinit ice lucid wait='0' atinit='zpcompinit'
zinit light zdharma/fast-syntax-highlighting

# 自动建议
zinit ice lucid wait="0" atload='_zsh_autosuggest_start'
zinit light zsh-users/zsh-autosuggestions

# 补全
zinit ice lucid wait='0'
zinit light zsh-users/zsh-completions

# 加载 OMZ 框架及部分插件
# zinit snippet OMZ::lib/completion.zsh
zinit snippet OMZ::lib/history.zsh
zinit snippet OMZ::lib/key-bindings.zsh
zinit snippet OMZ::lib/theme-and-appearance.zsh
zinit snippet OMZ::plugins/colored-man-pages/colored-man-pages.plugin.zsh
zinit snippet OMZ::plugins/git/git.plugin.zsh
# zinit snippet OMZ::themes/robbyrussell/robbyrussell.zsh-theme
# zinit snippet OMZ::plugins/z/z.plugin.zhs
# zinit snippet OMZ::plugins/sudo/sudo.plugin.zsh

# zinit ice svn
zinit snippet OMZ::plugins/extract/extract.plugin.zsh

zinit ice lucid wait='1'
zinit snippet OMZ::plugins/git/git.plugin.zsh

# 加载 pure 主题
# zinit ice pick"async.zsh" src"pure.zsh"
# zinit light sindresorhus/pure

# eval "$(starship init zsh)"

# 设置 zsh-autosuggestions 的提示配色以适配 Mariana 配色
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=white'

# llvm 配置
export PATH="/usr/local/opt/llvm/bin:$PATH"
export LDFLAGS="-L/usr/local/opt/llvm/lib"
export CPPFLAGS="-I/usr/local/opt/llvm/include"

```


