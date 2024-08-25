---
title: "配置 Sublime Text4为 C++ 编辑器"
date: 2023-02-06T09:06:25+08:00
lastmod: 2023-02-06T09:06:25+08:00 #更新时间
authors: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["sublime", "LSP", "tips"]
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---

## 概述
涉及以下插件的安装和配置`Package Control` `Terminus` `LSP` `LSP-clangd` `clang-format` `LSP-pyright` `LSP-json`

## 配置sublime
安装`Package Control`以进行包管理。

## Terminus
安装`Terminus`以实现sublime text4内的terminal。

绑定快捷键：
```json
[
	{
		"keys": [
			"ctrl+shift+t"
		],
		"command": "terminus_open",
		"args": {
            // 打开时要执行的命令
            // "cmd": "fish",
			"cwd": "${file_path:${folder}}"
		}
	}
]
```

自定义在`Terminus`的终端中编译运行cpp文件:

在`Tools->Build System->New Build System`中新建编译文件，保存为`CppTerminus.sublime-build`，替换内容为:
```json
{
	// MacOS
	"cmd": [
		"zsh",
		"-c",
		"clang++ '${file}' -std=c++17 -stdlib=libc++ -o '${file_path}/../bin/${file_base_name}' && ${file_path}/../bin/${file_base_name}"
	],
	"file_regex": "^(..{FNXX==XXFN}*):([0-9]+):?([0-9]+)?:? (.*)$",
	"working_dir": "${file_path}",
	"encoding": "utf-8",
	"selector": "source.c, source.c++",
	"variants": [
		{
			"name": "Run In Terminus",
			"target": "terminus_exec",
			"cancel": "terminus_cancel_build",
			"cmd": [
				"zsh",
				"-c",
				"clang++ '${file}' -std=c++17 -stdlib=libc++ -o '${file_path}/../bin/${file_base_name}' && ${file_path}/../bin/${file_base_name}"
			]
		},
		{
			"name": "Create Input File",
			"cmd": [
				"zsh",
				"-c",
				"touch ${file_path}/../in_out/${file_base_name}.in && open -a Sublime\\ Text ${file_path}/../in_out/${file_base_name}.in"
			]
		},
		{
			"name": "Run In Terminal",
			"cmd": [
				"zsh",
				"-c",
				"clang++ '${file}' -std=c++17 -stdlib=libc++ -o '${file_path}/../bin/${file_base_name}' && open -a Terminal.app '${file_path}/../bin/${file_base_name}'"
			]
		},
	]
}
```
注意要保证源文件和`bin`文件夹、`in_out`文件夹在同一目录下。

## 配置LSP + LSP-clangd

安装这两个插件，windows和linux需要手动安装`clangd`并添加到path。

### mac下安装clangd
我的mac已经自带了`clangd`，安装好这两个插件即可实现语法提示；
如果没有安装`clangd`，通过以下命令安装：
```shell
brew install llvm
echo 'export PATH="/opt/homebrew/opt/llvm/bin:$PATH"' >> ~/.zshrc
echo 'export LDFLAGS="-L/opt/homebrew/opt/llvm/lib"' >> ~/.zshrc
echo 'export CPPFLAGS="-I/opt/homebrew/opt/llvm/include"' >> ~/.zshrc
```

以上是针对默认 shell 为 zsh 的配置，如果你像我一样，mac 的默认 shell 是 fish，那么需要修改 `~/.config/fish/config.fish`，添加环境变量，添加下方法为追加以下内容：

```sh
set -gx PATH /opt/homebrew/opt/llvm/bin $PATH
set -gx LDFLAGS -L/opt/homebrew/opt/llvm/lib
set -gx CPPFLAGS -I/opt/homebrew/opt/llvm/include
```

### Debian(testing)安装clangd和clang
`sudo apt install clangd`, `sudo apt install llvm`, `sudo apt install clang`

### windows下安装llvm
借助scoop, 执行 `scoop install llvm`，然后安装**Visual Stdudio Build Tools**，鉴于windows上clang默认的c++库就是msvc，所以就用这个吧，别折腾`mingw`了。

### 配置LSP-clangd
到`Preferences->Package Settings->LSP->Settings`，写入这样几行
```json
{
    // 在主页面只显示error红色下划线
    //"show_diagnostics_severity_level": 1,
    // 代码提示显示灯泡图标
    "show_code_actions": "bulb",
    // 保存时自动格式化
    "lsp_format_on_save": true,
}
```

再到`Preferences->Package Settings->LSP->Servers->LSP-clangd`中，写入以下几行
```json
// Settings in here override those in "LSP-clangd/LSP-clangd.sublime-settings"
{
    "initializationOptions": {
        // 启用clang-tidy代码检查，可能启用后warning会比较多，自己看着办吧
        "clangd.clang-tidy": true,
        // 美化clangd输出的JSON
        "clangd.pretty": true,
    }
}
```

再到`project/code`（源文件所在目录）下新建`.clang-tidy`文件，写入:
```txt
Checks: "bugprone-*,\
google-*,\
misc-*,\
modernize-*,\
performance-*,\
readability-*,\
portability-*,\
"
HeaderFilterRegex: 'Source/cm[^/]*\.(h|hxx|cxx)$'
CheckOptions:
  - key:   modernize-use-default-member-init.UseAssignment
    value: '1'
  - key:   modernize-use-equals-default.IgnoreMacros
    value: '0'
  - key:   modernize-use-auto.MinTypeNameLength
    value: '80'
```

LSP-clangd默认是使用`c++98`来检查代码的，要修改为`c++17`，需要在项目根目录下新建`.clangd`文件，文件内容如下:

```txt
CompileFlags:
	Add: [-std=c++17]
```

更推荐方式为利用`cmake`生成`compile_commands.json`, `CMakeLists.txt`的内容如下：

```json
cmake_minimum_required(VERSION 3.22)
# Enable C++11
set(CMAKE_CXX_STANDARD 17)

# 设置项目名
project(LeetCpp)

# 源文件
aux_source_directory(. SOURCES)

# 头文件
include_directories(.)

add_executable(${PROJECT_NAME} ${SOURCES})
```

通过`cmake .. -DCMAKE_EXPORT_COMPILE_COMMANDS=1`生成`compile_commands.json`文件，注意要在`build`目录下，且`build`的上级目录存在`CMakelists.txt`。

## 配置`clang-format`
安装好`clang-format`插件之后，只需在`project/code`下新建`.clang-format`文件，写入以下内容

> 抄的网上的，修改了大括号换行的部分（左大括号不单独位于一行）

```sh
# 语言: None, Cpp, Java, JavaScript, ObjC, Proto, TableGen, TextProto
Language: Cpp
# BasedOnStyle: LLVM

# 访问说明符(public、private等)的偏移
AccessModifierOffset: -2

# 开括号(开圆括号、开尖括号、开方括号)后的对齐: Align, DontAlign, AlwaysBreak(总是在开括号后换行)
AlignAfterOpenBracket: Align

# 连续赋值时，对齐所有等号
AlignConsecutiveAssignments: false

# 连续声明时，对齐所有声明的变量名
AlignConsecutiveDeclarations: false

# 右对齐逃脱换行(使用反斜杠换行)的反斜杠
AlignEscapedNewlines: Right

# 水平对齐二元和三元表达式的操作数
AlignOperands: true

# 对齐连续的尾随的注释
AlignTrailingcomments: false

# 不允许函数声明的所有参数在放在下一行
AllowAllParametersOfDeclarationOnNextLine: false

# 不允许短的块放在同一行
AllowShortBlocksOnASingleLine: true

# 允许短的case标签放在同一行
AllowShortCaseLabelsOnASingleLine: true

# 允许短的函数放在同一行: None, InlineOnly(定义在类中), Empty(空函数), Inline(定义在类中，空函数), All
AllowShortFunctionsOnASingleLine: None

# 允许短的if语句保持在同一行
AllowShortIfStatementsOnASingleLine: true

# 允许短的循环保持在同一行
AllowShortLoopsOnASingleLine: true

# 总是在返回类型后换行: None, All, TopLevel(顶级函数，不包括在类中的函数), 
# AllDefinitions(所有的定义，不包括声明), TopLevelDefinitions(所有的顶级函数的定义)
AlwaysBreakAfterReturnType: None

# 总是在多行string字面量前换行
AlwaysBreakBeforeMultilineStrings: false

# 总是在template声明后换行
AlwaysBreakTemplateDeclarations: true

# false表示函数实参要么都在同一行，要么都各自一行
BinPackArguments: true

# false表示所有形参要么都在同一行，要么都各自一行
BinPackParameters: true

# 大括号换行，只有当BreakBeforeBraces设置为Custom时才有效
BraceWrapping:
  # class定义后面
  AfterClass: false
  # 控制语句后面
  AfterControlStatement: false
  # enum定义后面
  AfterEnum: false
  # 函数定义后面
  AfterFunction: false
  # 命名空间定义后面
  AfterNamespace: false
  # struct定义后面
  AfterStruct: false
  # union定义后面
  AfterUnion: false
  # extern之后
  AfterExternBlock: false
  # catch之前
  BeforeCatch: false
  # else之前
  BeforeElse: false
  # 缩进大括号
  IndentBraces: false
  # 分离空函数
  SplitEmptyFunction: false
  # 分离空语句
  SplitEmptyRecord: false
  # 分离空命名空间
  SplitEmptyNamespace: false

# 在二元运算符前换行: None(在操作符后换行), NonAssignment(在非赋值的操作符前换行), All(在操作符前换行)
BreakBeforeBinaryOperators: NonAssignment

# 在大括号前换行: Attach(始终将大括号附加到周围的上下文), Linux(除函数、命名空间和类定义，与Attach类似), 
#   Mozilla(除枚举、函数、记录定义，与Attach类似), Stroustrup(除函数定义、catch、else，与Attach类似), 
#   Allman(总是在大括号前换行), GNU(总是在大括号前换行，并对于控制语句的大括号增加额外的缩进), WebKit(在函数前换行), Custom
#   注：这里认为语句块也属于函数
BreakBeforeBraces: Custom

# 在三元运算符前换行
BreakBeforeTernaryOperators: false

# 在构造函数的初始化列表的冒号后换行
BreakConstructorInitializers: AfterColon

#BreakInheritanceList: AfterColon

BreakStringLiterals: false

# 每行字符的限制，0表示没有限制
ColumnLimit: 0

CompactNamespaces: true

# 构造函数的初始化列表要么都在同一行，要么都各自一行
ConstructorInitializerAllOnOneLineOrOnePerLine: false

# 构造函数的初始化列表的缩进宽度
ConstructorInitializerIndentWidth: 4

# 延续的行的缩进宽度
ContinuationIndentWidth: 4

# 去除C++11的列表初始化的大括号{后和}前的空格
Cpp11BracedListStyle: true

# 继承最常用的指针和引用的对齐方式
DerivePointerAlignment: false

# 固定命名空间注释
FixNamespacecomments: false

# 缩进case标签
IndentCaseLabels: false

IndentPPDirectives: None

# 缩进宽度
IndentWidth: 4

# 函数返回类型换行时，缩进函数声明或函数定义的函数名
IndentWrappedFunctionNames: false

# 保留在块开始处的空行
KeepEmptyLinesAtTheStartOfBlocks: false

# 连续空行的最大数量
MaxEmptyLinesToKeep: 1

# 命名空间的缩进: None, Inner(缩进嵌套的命名空间中的内容), All
NamespaceIndentation: None

# 指针和引用的对齐: Left, Right, Middle
PointerAlignment: Right

# 允许重新排版注释
Reflowcomments: false

# 允许排序#include
SortIncludes: false

# 允许排序 using 声明
SortUsingDeclarations: false

# 在C风格类型转换后添加空格
SpaceAfterCStyleCast: false

# 在Template 关键字后面添加空格
SpaceAfterTemplateKeyword: true

# 在赋值运算符之前添加空格
SpaceBeforeAssignmentOperators: true

# SpaceBeforeCpp11BracedList: true

# SpaceBeforeCtorInitializerColon: true

# SpaceBeforeInheritanceColon: true

# 开圆括号之前添加一个空格: Never, ControlStatements, Always
SpaceBeforeParens: ControlStatements

# SpaceBeforeRangeBasedForLoopColon: true

# 在空的圆括号中添加空格
SpaceInEmptyParentheses: false

# 在尾随的评论前添加的空格数(只适用于//)
SpacesBeforeTrailingComments: 1

# 在尖括号的<后和>前添加空格
SpacesInAngles: false

# 在C风格类型转换的括号中添加空格
SpacesInCStyleCastParentheses: false

# 在容器(ObjC和JavaScript的数组和字典等)字面量中添加空格
SpacesInContainerLiterals: true

# 在圆括号的(后和)前添加空格
SpacesInParentheses: false

# 在方括号的[后和]前添加空格，lamda表达式和未指明大小的数组的声明不受影响
SpacesInSquareBrackets: false

# 标准: Cpp03, Cpp11, Auto
Standard: Cpp11

# tab宽度
TabWidth: 4

# 使用tab字符: Never, ForIndentation, ForContinuationAndIndentation, Always
UseTab: Never
```

## 命令行使用sublime text4打开文件
mac下添加软连接: `ln  /Applications/Sublime\ Text.app/Contents/SharedSupport/bin/subl /usr/local/bin/subl`

debian下执行`sudo ln -s /opt/sublime_text/sublime_text /usr/local/bin/subl`

之后就能用`subl test.cpp`命令来调用sublime text4打开`test.cpp`了。

## leetcode相关配置
安装leetgo: `brew install j178/tap/leetgo`

在预期的leetcode项目目录下执行`leetgo init`，然后配置`leetgo.yaml`
```yaml
author: zwyyy456
language: en
code:
  lang: cpp
  filename_template: '{{ .Id}}{{ if .SlugIsMeaningful }}.{{ .Slug }}{{ end }}'
leetcode:
  site: https://leetcode.com
#  credentials:
#    from: browser
editor:
  use: custom
  command: "subl"
  args: "{{.Folder}}"
  # use: vscode
```

## 自定义字体

在 Sublime Text 的配置文件中（`Preference` -> `Settings`），添加 `"font_face": "Name of Font"` 即可，例如我这里使用的是 `"font_face": "Cascadia Mono NF",`，要注意的是，有些字体例如 `FiraMono Nerd Font Mono`，直接将 `font_face` 的值设置为它在系统中显示的字体名称，Sublime Text 会显示无法找到该字体转而使用系统默认字体例如 `Consolas`，我们需要将字体上传到 [opentype.js](https://opentype.js.org/font-inspector.html)，再查看 `Naming table` -> `postScriptName` 中显示的值，将 `font_face` 的值设为该值，例如 `FiraMono Nerd Font Mono` 显示的值为 `FiraMonoNFM-Regular`。

> 奇怪的地方在于，对于 `Cascadia Mono NF`，将 `font_face` 的值设置为 `postScriptName` 显示的值反而会有问题。

完整的配置文件如下：
```json
{
	"ignored_packages":
	[
		"Vintage",
	],
	"vintageous_ctrl_keys": true,
	"vintageous_i_escape_jk": true,
	"vintageous_use_sys_clipboard": true,
	"neovintageous_build_version": 13200,
	"font_face": "Cascadia Mono NF",
	"font_size": 12,
	"relative_line_numbers": true,
}
```

## 启用 Vim 模式

利用 Sublime Text4 的 package control 安装 NeoVintageous 插件。

### 启用`ctrl + [`作为`esc`
点击`sublime Text -> settings ->settings`，编辑右侧的配置文件，添加`"vintageous_ctrl_keys": true,`。

### 启用`jk`为`esc`
点击`sublime Text -> settings ->settings`，编辑右侧的配置文件，添加`"vintageous_i_escape_jk": true,`

### 使`yy`会复制到系统剪贴板
点击`sublime Text -> settings ->settings`，编辑右侧的配置文件，添加`"vintageous_use_sys_clipboard": true,`








