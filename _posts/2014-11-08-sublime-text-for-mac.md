---
layout: post
title: Sublime Text for Mac 配置指南
category: Program

---

Mac OS X 从不缺乏优秀的代码编辑器，从 Vim 和 Emacs 到 Xcode、Eclipse、PyCharm，都有着为数甚多的忠实用户。然而大多数的代码编辑器都是为软件工程而设计的，若只是写写算法题，那么只需要简单的命令行程序，用这些面向工程的 IDE 堪比杀鸡用牛刀了。

Sublime Text 作为一个轻量级的代码编辑器，对于单文件编程非常友好，加上其通用性、易扩展性和干净的界面，是一个不错的选择。

即便如此，Sublime Text 还是有一些不尽如人意的地方需要进一步配置，以下便是 C / C++、Python、HTML 的几处配置技巧。

测试环境：`Sublime Text 3 (Build 3065) on OS X Yosemite (10.10)`

<!--more-->

## C / C++

在 Sublime Text 中，预设的 Build System 只有 C++ 而没有 C，虽然没什么大碍但会弹出警告 `this behavior is deprecated`；最不科学的地方是，下方内置的控制台竟然不能输入数据。

![](/images/sublime-text-for-mac-00.png)  解决这两个问题需要创建两个自定义的 Build System，分别用于 C 和 C++，并设置其在运行程序时不使用内置控制台，而是直接调用终端运行。找到 `Tools > Build System > New Build System...`，将其命名为`Clang.sublime-build`，内容为：

    {
        "cmd": ["clang", "${file}", "-o", "${ file_path }/${ file_base_name }"],
        "selector": "source.c",
        "variants": [
            { "name": "Run",
              "cmd": ["bash", "-c", "clang '${file}' -o '${file_path}/${file_base_name}' && open -a Terminal.app '${file_path}/${file_base_name}'"]
            }
        ]
    } 
![](/images/sublime-text-for-mac-01.png) 
实际上这就是一个 JSON 文件，相关资料可以参阅 [Sublime Text Unofficial Documentation](http://docs.sublimetext.info/en/latest/reference/build_systems.html)。相应地，C++ 的配置文件将所有的 `Clang` 替换为 `Clang++` 即可。 
## Python

Python 也存在上述的内置控制台无法输入的问题，这时有比自己重写 Build System 更方便的解决方案。[GitHub](https://github.com/wuub/SublimeREPL) 上有一个项目 **SublimeREPL**，支持在 Sublime 中运行交互式开发环境（即 REPL）。

首先需要安装 Package Control，这是 Sublime Text 的插件包管理器，安装方法见 [Installation - Package Control](https://sublime.wbond.net/installation)。接着按 `⇧⌘P` 调出 `Command Palette`，键入「install」打开 `Package Control: Install Package`，找到 `SublimeREPL` 即可安装。

![](/images/sublime-text-for-mac-02.png)

安装完成后，可以通过 `Tools > SublimeREPL > Python > Python` 在新窗口中打开交互式开发环境，或是通过同菜单下的 `Python - RUN current file` 运行当前文件。除此之外，也有其他简便的办法，其一是按 `⇧⌘P` 并输入「python」，可以在列表中看到 SublimeREPL 的相关命令。其二是为这些命令设置快捷键，这里以 `Python - RUN current file` 为例，为其跟 Python IDLE 相同的快捷键 `F5`。

打开 `Sublime Text > Preferences > Key Bindings - User`，在文件中输入：

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

![](/images/sublime-text-for-mac-03.png)

同样地，**Ruby** 等脚本语言的代码也能通过 SublimeREPL 运行了。

## HTML

![](/images/sublime-text-for-mac-04.png)

若是做网页设计，当然有 [Coda 2](http://www.panic.com/coda/)、[WebStorm](http://www.jetbrains.com/webstorm/)、[Brackets](http://brackets.io) 这些更专业的选择，不过用 Sublime Text 写 HTML / CSS 也未尝不可。ST 本身支持 HTML 代码着色、补全结束标签，但不具备预览网页的功能，我们仍可以照 C / C++ 的办法自定义 Build System。如果这样写 `HTML.sublime-build`：

    {
        "cmd": ["open", "-a", "Safari.app", "${file}"],
        "selector": "source.html",
        "variants": [
            { "name": "Run",
              "cmd": ["open", "-a", "/Applications/Google Chrome.app", "${file}"]
            }
        ]
    }

![](/images/sublime-text-for-mac-05.png)

按下 `⌘B` 和 `⇧⌘B` 会分别在 Safari 和 Chrome 中打开当前 HTML 文件，这样便可以快捷地预览设计中的页面。

以上便是对 Sublime Text 配置的个人经验，希望对大家有所帮助。
