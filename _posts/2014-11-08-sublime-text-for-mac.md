---
layout: post
title: Sublime Text for Mac 使用配置
category: Tech
---

Mac OS X 从不缺乏优秀的代码编辑器，从 Vim 和 Emacs 到 Xcode、Eclipse、IntelliJ IDEA，都有着为数甚多的忠实用户。然而大多数的代码编辑器都是为软件工程而设计的，若只是写写算法题，那么只需要简单的命令行程序，用这些面向工程的 IDE 堪比杀鸡用牛刀了。

Sublime Text 作为一个轻量级的代码编辑器，对于单文件编程非常友好，原生支持不少主流语言的编译运行。它吸收了前辈 TextMate 的优点，并在可扩展性方面更胜一筹。

即便如此，Sublime Text 还是有一些不尽如人意的地方需要进一步配置，以下便是 C/C++、Python、Ruby、HTML 的几处配置技巧。

测试环境：`Sublime Text 3 (Build 3065) on OS X Yosemite (10.10)`

<!--more-->

## C / C++

在 Sublime Text 中最不科学的地方是，下方内置的控制台竟然不能输入数据，即程序无法完成正常的 stdin 输入。其次预设的 Build System 只有 C++ 而没有 C，虽然没什么大碍但会弹出警告「this behavior is deprecated」。

![](/images/sublime-text-for-mac-00.png)

 解决这两个问题需要创建自定义 Build System，设置其在运行程序时不使用内置控制台，而是直接调用终端运行。我们可以修改内置的 Package，也可以直接点击 `Tools > Build System > New Build System...` 新建自己的配置文件。C++ 的包位于 `/Applications/Sublime Text.app/Contents/MacOS/Packages/C++.sublime-package`，它可以作为归档文件打开，修改其中的 `C++.sublime-build` 即可。修改后的内容为：

```json
{
    "shell_cmd": "g++ \"${file}\" -o \"${file_path}/${file_base_name}\"",
    "file_regex": "^(..[^:]*):([0-9]+):?([0-9]+)?:? (.*)$",
    "working_dir": "${file_path}",
    "selector": "source.c++",

    "variants":
    [
        {
            "name": "Run",
            "shell_cmd": "g++ \"${file}\" -o \"${file_path}/${file_base_name}\" && open \"${file_path}/${file_base_name}\" -a Terminal.app"
        }
    ]
}
```

实际上这就是一个 JSON 文件，相关资料可以参阅 [Sublime Text Unofficial Documentation](http://docs.sublimetext.info/en/latest/reference/build_systems.html)。相应地，C 的配置文件将所有的 `g++` 或 `clang++` 替换为 `gcc` 或 `clang` 即可。 

## Python

Python 也存在上述的内置控制台无法输入的问题，这时有比自己重写 Build System 更方便的解决方案。[GitHub](https://github.com/wuub/SublimeREPL) 上有一个项目 **SublimeREPL**，支持在 Sublime 中运行交互式开发环境（即 REPL）。

首先需要安装 Package Control，这是 Sublime Text 的插件包管理器，安装方法见 [Installation - Package Control](https://sublime.wbond.net/installation)。接着按 `⇧⌘P` 调出 `Command Palette`，键入「install」打开 `Package Control: Install Package`，找到 `SublimeREPL` 即可安装。

![](/images/sublime-text-for-mac-01.png)

安装完成后，可以通过 `Tools > SublimeREPL > Python > Python` 在新窗口中打开交互式开发环境，或是通过同菜单下的 `Python - RUN current file` 运行当前文件。除此之外，也有其他简便的办法，其一是按 ⇧⌘P 并输入「python」，可以在列表中看到 SublimeREPL 的相关命令。其二是为这些命令设置快捷键，这里以 `Python - RUN current file` 为例，为其跟 Python IDLE 相同的快捷键 `F5`。

打开 `Sublime Text > Preferences > Key Bindings - User`，在文件中输入：

```json
[
    {
        "keys": ["f5"],
        "caption": "SublimeREPL: Python - RUN current file",
        "command": "run_existing_window_command",
        "args": {
            "id": "repl_python_run",
            "file": "config/Python/Main.sublime-menu"
        }
    }
]
```

![](/images/sublime-text-for-mac-02.png)

## Ruby

SublimeREPL 当然也支持 Ruby，不过非常坑爹的是，它不提供运行当前文件的功能，只提供交互式界面，而在高版本的 Pry 下默认的这个交互式界面还没法用。

还好 GitHub 上已经有人提出了修复的方法：[Fix ruby Pry helper for Pry version >= 0.10.0](https://github.com/wuub/SublimeREPL/pull/372)。至于运行单个文件的功能，还是需要自己添加。打开文件`~/Library/Application Support/Sublime Text 3/Packages/SublimeREPL/config/Ruby/Main.sublime-menu`，在与 `Ruby - IRB (deprecated)` 同级的位置加入以下代码：

```json
{
    "command":"repl_open",
    "caption":"Ruby - RUN current file",
    "id":"repl_ruby_run",
    "mnemonic":"r",
    "args": {
        "type":"subprocess",
        "external_id":"ruby",
        "encoding":"utf8",
        "cmd":["ruby", "-w", "$file_basename"],
        "soft_quit":"\nexit\n",
        "cwd":"$file_path",
        "cmd_postfix":"\n",
        "syntax":"Packages/Ruby/Ruby.tmLanguage"
    }
}
```

同样地，可以在 `Key Bindings - User` 为其设置快捷键。

## HTML

若是做 Web 开发，当然有 [Brackets](http://brackets.io)、[Coda](http://www.panic.com/coda/)、[Espresso](http://www.macrabbit.com/espresso/)、[WebStorm](http://www.jetbrains.com/webstorm/) 这些更专业的选择，不过用 Sublime Text 写 HTML / CSS 也算凑合。ST 本身支持 HTML 代码着色、补全结束标签，但不具备预览网页的功能，我们仍可以照 C / C++ 的办法自定义 Build System。如果这样写 `HTML.sublime-build`：

```json
{
    "cmd": ["open", "-a", "Safari.app", "${ file }"],
    "selector": "source.html",
    "variants": [
        { "name": "Run",
          "cmd": ["open", "-a", "/Applications/Google Chrome.app", "${ file }"]
        }
    ]
}
```

按下 `⌘B` 和 `⇧⌘B` 会分别在 Safari 和 Chrome 中打开当前 HTML 文件，这样便可以快捷地预览设计中的页面。

以上便是对 Sublime Text 配置的个人经验，希望对大家有所帮助。
