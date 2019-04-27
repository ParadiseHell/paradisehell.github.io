---
layout:     post
title:      "如何更好的使用命令行"
subtitle:   "一起来抛弃上下左右键"
date:       2019-04-26
author:     "ChengTao"
header-img: "img/linux.jpg"
tags:
    - Lunix
---

> 这是一篇纯翻译的文章，旨在让更多的人更好使用命令行。

原文章地址：<a href="https://www.howtogeek.com/howto/ubuntu/keyboard-shortcuts-for-bash-command-shell-for-ubuntu-debian-suse-redhat-linux-etc/" target="_blank">The Best Keyboard Shortcuts for Bash (aka the Linux and macOS Terminal)</a>

## 系统环境说明
macOs Mojave 10.14.4

## 游标移动
- Ctrl+A : 将游标移动到起始位置。
- Ctrl+E : 将游标移动到结束位置。
- Alt+B : 将游标移动到前一个单词的起始位置（如果当前游标不在单词的尾部，则将游标移动到当前单词的起始位置）。
- Ctrl+B : 将游标向前移动一个字符。
- Alt+F : 将游标移动到后一个单词的起始位置。
- Ctrl+F : 将游标向后移动一个字符。
- Ctrl+XX : 将游标在当前位置和起始位置之间移动。这将允许你按 Ctrl+XX 回到起始位置，然后你就可以在起始位置做出一些修改，然后当你再按 Ctrl+XX 后，游标将回到最原始的游标的位置。使用这个快捷键你需要按住 Ctrl 键然后连续按两次 X 键。

## 文本删除
- Ctrl+D : 删除当前游标下的一个字符。
- Alt+D : 删除当前游标所在单词后面的所有字符。
- Ctrl+H : 删除当前游标前的一个字符。

## 排版修复
- Alt+T : 交换将当前游标下的单词和前一个单词。
- Ctrl+T : 交换将当前游标下的字符和前一个字符。
- Ctrl+- : 撤回上一个按键操作，并且可以多次执行。举例：输入 cd, 然后按 Ctrl+-, 就会变成 c, 再按一下 Ctrl+-, cd 就没有了。

## 剪贴和复制
- **Ctrl+W** : 剪切在当前游标前的单词，并将剪切的内容添加到剪切板。
- **Ctrl+K** : 剪切在当前游标后的所有字符，并将剪切的内容添加到剪切板。
- **Ctrl+Y** : 从剪切板粘贴最后剪切的内容。

## 大写字符
- **Alt+U** : 大写从当前游标所在位置到当前游标所在单词的最后位置的所有字符。
- **Alt+L** : 小写从当前游标所在位置到当前游标所在单词的最后位置的所有字符。
- **Alt+C** : 大写从当前游标下的字符，并将游标移到当前单词的末尾。

## 命令行历史
- **Ctl+P** : 从命令历史中回到前一个命令，多次执行将会遍历命令历史，相当于执行了 Up 键。
- **Ctl+N** : 从命令历史中回到下一个命令，多次执行将会遍历命令历史，相当于执行了 Down 键。
- **Ctl+R** : 恢复命令如果你是从命令历史中拉去的命令并且修改过该命令。
