---
title: "vscode 使用 clangd"
date: 2023-03-23T15:56:21+08:00
lastmod: 2023-03-23T15:56:21+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tutorial"]
tags: ["geek", "clangd", "vscode", "tips"]
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
## 环境要求
使用 **wsl** 或者 **macOS**，**Linux** 下同理，暂时不考虑纯 winodws。

以 **wsl** 为例，执行以下指令

```bash
sudo apt install clang clangd lldb cmake
```

**macOS**如果安装过**xcode**工具包，就附带了Apple Clang编译器，否则执行`brew install llvm`，然后输入以下指令添加环境变量

```sh
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

然后在vscode中安装 **CodeLLDB**、**clangd**、**Cmake**、**Xmake**、**Cmake Tools** 这几个插件，其中除了**Xmake**之外都必须安装。

## 开始

随便找一个空文件打开，然后`Ctrl+shift+p`（macOS下为`cmd+shift_p`）打开下拉菜单，搜索`camke`，选择**Quick start**：
之后给项目起个名字，如`webserver`，输出类型选`Executable`而不是`lib`，**Kit**选择**Clangxxxxx-gnu**。

macOS 必须选择 homebrew 安装的 clang kit，否则会出现无法找到 `ninja` 的问题。

这里的话其实是 Cmake 插件给你默认生成了 `CMakeLists.txt` 并生成了对应的 `compile_commands.json` 文件，建议根据自己的项目结构（便于 `include` 等）自己写 `CMakeLists.txt`。

然后需要让**clangd**识别项目的编译数据库，在**vscode**设置中搜索**clangd**，在`Arguments`配置项中输入参数`--compile-commands-dir=${workspaceFolder}/build`，然后点击确定。

## 运行与调试

点击`build`生成的可执行文件就位于`build`目录下，按下`F5`开始调试，会报错并在项目根目录下生成`.vscode`目录以及`.vscode/launch.json`文件，编辑这个`json`文件，将`program`那一栏修改为`"program": "${workspaceFolder}/build/${workspaceFolderBasename}"`，之后就能按`F5`调试了。

注意`cmake`默认只会编译`main.cpp`这一源文件，修改了源文件或者添加断点，要重新执行**cmake**插件的`build`之后才能再调试。

或者参照[vscode文档](https://code.visualstudio.com/docs/editor/debugging)配置`lauch.json`文件。

## 代码格式化
在项目根目录下创建`.clang-format`文件，编辑内容为
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

## clang-tidy
在项目根目录下创建`.clang-tidy`文件，内容修改为
```
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

## 参考内容

- [几乎无痛的VSCode+clangd+lldb+cmake配置C/C++开发环境指南](https://zhuanlan.zhihu.com/p/566365173)

- [配置sublime text4为C++编辑器](https://blog.zwyyy456.tech/zh/posts/blog/sublime_cpp/)
