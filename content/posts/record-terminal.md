---
title: "Record Terminal as GIF"
date: 2021-11-07T21:35:44+08:00
categories:
  - Code
draft: false
---

这几天在做一个 [CLI 项目](https://github.com/iamgodot/py-tldr)，因为涉及到命令行操作，所以想录制一段 GIF 放在 README 中展示。

印象里这种工具都是 JS 写的，但搜了搜居然发现有个 Python 的实现：[asciinema](https://github.com/asciinema/asciinema)

用法很简单，就是安装好之后在命令行执行 `asciinema rec` 便会启动一个新的 Shell 并开始录制，录制的时候就像正常使用 Terminal 一样即可，完成之后按 Ctrl-D 或者 Exit 退出，asciinema 会把录制好的 cast 文件保存到本地，也可以选择上传到他们的网站：asciinema.org.

那么 cast 文件是什么，又怎样得到 GIF 呢？其实这是 asciinema 自己定义的一种文件格式：

> A CAST file is a record of a [terminal](https://techterms.com/definition/terminal) session recorded using asciinema, an [open source](https://techterms.com/definition/opensource) terminal recording program. It contains a [JSON-formatted](https://techterms.com/definition/json) header and a timestamped record of the text typed and printed during a terminal session.

文件内容大概是这个样子的：

```
{"version": 2, "width": 124, "height": 63, "timestamp": 1636292616, "env": {"SHELL": "/usr/bin/zsh", "TERM": "screen-256color"}}
[1.538309, "o", "#"]
[1.915683, "o", "\b# "]
[2.335422, "o", "l"]
[2.405338, "o", "e"]
[2.524792, "o", "t"]
[3.010565, "o", "'"]
[3.13154, "o", "s"]
```

就像描述里说的，第一行是 JSON 格式的 header，里面存放了一些 Terminal 的元信息，比如尺寸和环境变量。下面的每一行数据都包含了时间戳和对应的终端打印信息，也就是在录制的时候键入的命令以及输出。

对于 cast 文件，还可以直接用 asciinema 在终端回放录制的内容，比如 `asciinema play -i 2 demo.cast` 可以以两倍速播放。

从官方 Issue 看，asciinema 本身目前还不支持直接导出 GIF 文件，不过其他的工具应运而生，比如：[asciicast2gif](https://github.com/asciinema/asciicast2gif).

这个工具主要是用 JS 来解析 cast 文件并且生成若干张 PNG 图片，再调用 ImageMagick 和 gifsicle 这两个工具合成出最终的 GIF 文件：

> `asciicast2gif` shell script parses command line arguments and executes Node.js script (`main.js`). `main.js` loads asciicast (either from remote URL or local filesystem), generates text representation of the screen for each frame using [asciinema-player](https://github.com/asciinema/asciinema-player)'s virtual terminal emulator, and sends it to PhantomJS-based renderer script (`renderer.js`), which saves PNG screenshots to a temporary directory. Finally, `main.js` calls ImageMagick's `convert` on these PNG images to construct GIF animation, also piping it to `gifsicle` to get the final, optimized GIF file.

我不想在本地用 NPM 安装，所以选择了他们的 Docker 镜像试用，结果却没有想象中的顺利。原因大概是录制的时间较长（其实也就几分钟），生成的 cast 文件有 27K 左右，尝试渲染之后报错。在 Issue 查到是由于图片数量太多，gifsicle 合成失败。

不成熟的工具用起来就是不顺手，正在纠结要不要进坑，又发现了一个在线转换 cast 为 gif 的网站：[gifcast](https://dstein64.github.io/gifcast/).

试了一下异常顺利，而且网页的好处是可以预览各种 Theme 的效果，如下图：

![](https://static.iamgodot.com/content/images/2021-11-07_22-13.png)

可能是 GIF 比较大，直接把图床的 Link 放到 README 里面加载得太慢，于是我还是选择把文件放到 REPO 里面并用 Relative Path 的方式引用。

之后在 GitHub 上面打开[展示的效果](https://github.com/iamgodot/py-tldr)还是不错的，这里贴一下：

![](https://static.iamgodot.com/content/images/tldr.gif)

---

*References*

- https://github.com/asciinema/asciinema
- https://dstein64.github.io/gifcast/
