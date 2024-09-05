---
title: "vscode 使用 vscode-neovim 插件"
date: 2023-11-08T16:52:21+08:00
lastmod: 2023-11-08T16:52:21+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["setup"]
tags: ["nvim", "vscode"]
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
## 启用插件

### Win11

插件设置：`Settings-user`页面，`Neovim Executable Paths: Win32`内容设置为`C:\Users\zwyyy\scoop\apps\neovim\current\bin\nvim.exe`，`Neovim Init Vim Paths: Win32`内容无需设置。

### WSL

插件设置：`Settings-Remote [WSL:Debian]`页面，`Neovim Executable Paths: Linux`内容设置为`/usr/bin/nvim`，`Neovim Init Vim Paths: Linux`内容无需设置。

### MacOS

插件设置：`Settings-user`页面，`Neovim Executable Paths: Darwin` 内容设置为 `/opt/homebrew/bin/nvim`，`Neovim Init Vim Paths: Darwin` 内容无需设置。


**key bindings** 中查找`ctrl+c`，删除与`neovim`有关的两项，否则会导致插入模式下无法使用`ctrl+c`复制（我没有选择删除）。

## 设置 `jk` 为 `esc`

### 已于 1.11.1 版本废弃

编辑 `keybings.json`，添加

```json
{
    "command": "vscode-neovim.compositeEscape1",
    "key": "j",
    "when": "neovim.mode == insert && editorTextFocus",
    "args": "j"
},
{
    "command": "vscode-neovim.compositeEscape2",
    "key": "k",
    "when": "neovim.mode == insert && editorTextFocus",
    "args": "k"
}
```

## 使用 gc 作为注释快捷键

### 已废弃
原先解决方案是在 neovim 的配置目录的 `lua/vscode/config` 目录下新建 `keymaps.lua` 文件，内容为：

```lua
local keymap = vim.keymap
-- local opts = { noremap = true, silent = true }
-- vscode.neovim用于注释代码的按键映射
keymap.set("v", "gc", "<Plug>VSCodeCommentary")
keymap.set("n", "gc", "<Plug>VSCodeCommentary")
keymap.set("o", "gc", "<Plug>VSCodeCommentary")
keymap.set("n", "gcc", "<Plug>VSCodeCommentaryLine")
```

同时修改`init.lua`，添加`require("vscode.config.keymaps")`。

注意的是，这一方案随着 vscode 的 vscode.neovim 插件的更新已经失效。需要利用 neovim 中的 comment 插件来实现。

### comment.nvim

利用 `comment.nvim` 来实现 vscode 的调用 `gc` 快捷键注释。

文件结构目录如下：

```txt
├── LICENSE
├── README.md
├── init.lua
├── lazy-lock.json
├── lazyvim.json
├── lua
│   ├── config
│   │   ├── autocmds.lua
│   │   ├── keymaps.lua
│   │   ├── lazy.lua
│   │   └── options.lua
│   ├── plugins
│   │   ├── colorscheme.lua
│   │   ├── disabled.lua
│   │   └── example.lua
│   └── vscode
│       ├── config
│       │   ├── keymaps.lua
│       │   ├── lazy.lua
│       │   └── options.lua
│       └── plugins
│           ├── comment.lua
│           └── disabled.lua
└── stylua.toml
```

`init.lua` 的内容如下：

```lua
if vim.g.vscode then
  -- if true then
  -- Vscoe extension
  require("vscode.config.options")
  require("vscode.config.keymaps")
  -- require("vscode.init")
  require("vscode.config.lazy")
  -- require("vscode.plugins.disabled")
else
  -- bootstrap lazy.nvim, LazyVim and your plugins
  require("config.lazy")
end
```

`lua/vscode/config/lazy.lua` 的内容如下：

```lua
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", -- latest stable release
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

require("lazy").setup("vscode.plugins")
```

由于 `vscode.neovim` 是使用本机的 neovim 作为后端的，`lua/config/lazy.lua` 这一段代码就是让 `vscode.neovim` 也能使用 `lazy.nvim` 来管理插件，并且能够启用 `lua/vscode/plugins` 中的插件。

`lua/vscode/plugins/comment.lua` 的内容如下：

```lua
return {
  {
    "numToStr/Comment.nvim",
    lazy = false,
  },
}
```

`lua/vscode/config/lazy.lua` 与 `lua/vscode/plugins/comment.lua` 配合起来即通过 `lazy.nvim` 插件管理并启用了 `Comment.nvim` 插件，之后 vscode 在 normal 模式下也可以通过 `gc` 或者 `ngc`（其中 n 表示数字）来注释一行或者多行了。

同时 `lua/vscode/plugins/disabled.lua` 在 vscode 中禁用了 neovim 的 `tokyonight` 主题，防止 vscode 的代码高亮出现异常。

## 同步系统剪贴板

`lua/vscode/config/options.lua` 中内容修改为：

```lua
-- clipboard
local opt = vim.opt
opt.clipboard = "unnamedplus" -- Sync with system clipboard
```

## 与 LazyVim 的融合

LazyVim 有一个 extras 机制，打开 nvim，输入 `:LazyExtras`，根据提示勾选 `vscode` 即可。

这样，`init.lua` 可以写成

```lua
require("config.lazy")
if vim.g.vscode then
else
    require("config.osc52")
end
```

也可以实现 `kgc` 来注释多行代码。

但是依旧存在问题，此种模式虽然 VsCode 不再会出现高亮异常问题，但是 vscode-neovim 的 `<C-w>(h/j/k/l)` 会被 LazyVim 插件本身占用。

## 使用 neovim 的一些插件

`Comment.nvim` 已安装，还需要安装 `flash.nvim`，以及 `nvim-treesitter`、`nvim-treesitter-textobjects` 和 `nvim-ts-context-commentstring` 即可。

见配置文件仓库。

## 总结

配置文件已存放于远程仓库 [neovim_config](https://github.com/zwyyy456/nvim_config/tree/main)

