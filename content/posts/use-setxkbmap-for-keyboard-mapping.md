---
title: Use setxkbmap for Keyboard Mapping
date: 2021-10-16T04:57:10+08:00
categories:
  - Code
draft: false
---

这是一篇使用（过） Linux Xorg without any DE 以及笔记本外接键盘的人才能理解其中辛酸苦痛的抒情科普文。

# TL;DR

1. 外接键盘不要开蓝牙，直接插线，除非你乐意折腾
2. 使用 setxkbmap 而不是 xmodmap 来做 key remapping
3. 不要用 Xorg

# The Irrational Part

上午十点零一分，端坐到显示器前，深呼吸两次。

Ok，回滚内核版本之后 Screen Lock 卡死的问题果然消失了，我们继续。

进入系统，先看一下昨晚的日志吧：

```
...
需求目标：

1. 交换 L-Ctrl & Caps-Lock
2. 交换 Win & Alt

当前方案：

使用 xmodmap 自定义配置文件，在 startx 时执行

已知问题：

1. 系统挂起或蓝牙睡眠导致的 keyboard reconnection 会 reset 掉 Win&Alt 的交换
2. 之后重新执行 xmodmap 会导致 L-Ctrl&Caps-Lock 交换回原状，必须执行两次

后续跟进：
1. TTY console switch 快捷键不可用
2. Terminate X 快捷键 CAB(Ctrl-Alt-Backspace) 不可用
3. Keyboard backlight
...
```

唔，大概记起来了，昨晚改用了 setxkbmap 之后应该可以解决重连的问题了，不过奇怪的是 ctrl:swapcaps 的 option 一直没有生效，折腾到半夜也没好..

难道是有什么地方又去执行 xmodmap 了？奇怪，在 .xinitrc 里面已经把判断代码注释掉了呀，不管了，删除 ~/.Xmodmap 再说，反正是个软链接。

哈，居然成功了！看来就是这个原因，估计这也是已知问题 1 的根源，xmodmap 的设置被 setxkbmap 重置了，果然直接用后者是明智的。

不过可惜之前查了半天才解决的 .xinitrc 里面 xmodmap 执行顺序问题了：

```bash
...
if [ -f $HOME/.Xmodmap ]; then
    (sleep 1; xmodmap $HOME/.Xmodmap) &
fi

exec i3
...
```

虽然多花了 1s 的开机时间，但毕竟规避掉了 xmodmap 设置被 wm 莫名其妙 reset 的难点，不过现在也不需要了，唔，白搞了半天..

Setxkbmap 的 rule setting 就是好，这下插拔外接键盘 USB 也不会 reset 改键映射了。不过如果用蓝牙的话好像还不够，至少得配置 udev rule 开启蓝牙键盘唤醒再 run 个 remapping 的脚本才好，如此剩下就是背光了...

咦，好像又过去两个小时了？

...

站起来，默默找出键盘连接线，插好，保存唯一的 setxkbmap 配置，重启。

进入系统，改键映射正常，拔开 USB，重插，一切正常。唔，背光真好看。

看下时间，五分钟。唔，终于清静了。

# The Rational Part

## Keyboard mapping

用惯 Vim 的人可能都会有改键的需求，L-Ctrl <-> Caps-Lock 或者 Escape <-> Caps-Lock 等等。

另外 Mac 上面的 Option | Command 对应到 Windows 键盘是 Win | Alt, 我的习惯是把 Alt 当 Option 用，所以要换到左边去。

## About keyboard

键盘的原理大概是每个按键都有对应的（十六进制）scancode，在内核层 scancode 会被映射会 keycode，而在 X 中（比如 xmodmap）会进一步把 keycode 对应到不同的 keysym，比如 `keycode 38 = a A` 就代表 keycode 为 38 的键会输出字母 a，如果配合 Shift 键则输入 A.

而 xmodmap 改键的原理也是通过修改 keycode 的对应关系来实现的，只是在配置文件中还可以使用更方便的命令比如 clear/remove/add 来描述。

另外 xmodmap 还支持通过修改 pointer 参数来支持触摸屏滚动的方向，比如像 Mac 一样，手指上滑让页面下翻。

但是 xmodmap 只适合做比较简单的自定义，毕竟写起来复杂，还容易被 setxkbmap 覆盖。

## Use setxkbmap

setxkbmap 使用起来很简单，如果实现我自己的需求，想要开机生效的话，直接在 .xinitrc 中加入一行命令即可：

```bash
setxkbmap -option ctrl:swapcaps,altwin:swap_lalt_lwin
```

如果支持插拔的话，则要给 Xorg 增加配置文件（如 /etc/X11/xorg.conf.d/00-keyboard.conf）：

```
Section "ServerFlags"
	Option "DontZap" "False"
EndSection

Section "InputClass"
        Identifier "system-keyboard"
        MatchIsKeyboard "on"
        Option "XkbOptions" "ctrl:swapcaps,altwin:swap_lalt_lwin,terminate:ctrl_alt_bksp"
EndSection
```

## CAB

那么上面配置文件里面的 ServerFlags 和 option 中的 terminate:ctrl_alt_bksp 是做什么的？

就是来支持 CAB(Ctrl-Alt-Backspace) 快捷键关闭当前 X 程序的。这很有用，尤其是在 X window 卡死没有响应的时候。

当然 Ctrl-Alt-Fx 切换 tty 或者强制关机也没什么不可以的

---

WM 也好，XDG 的那些 DE 也好（GNOME/KDE/...），X 战警们的难题永无止境，然而时间有限，不得不做取舍的我只好以此纪念无辜逝去的时间以及无数个奋战到深夜的自己。

---

*References*

- https://wiki.archlinux.org/title/Xmodmap
- https://wiki.archlinux.org/title/Xorg/Keyboard_configuration
- http://www.linuxintro.org/wiki/Understand_how_keyboards_work