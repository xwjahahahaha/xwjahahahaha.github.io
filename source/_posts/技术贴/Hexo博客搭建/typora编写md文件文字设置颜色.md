---
title: typora编写md文件文字设置颜色
tags:
  - markdown
categories:
  - markdown
toc: true
date: 2020-10-13 13:09:06
---

# typora编写md文件文字设置颜色

AutoHotKey是一款著名的windows系统快捷键设置的软件，轻便小巧。

官方下载: https://autohotkey.com/download/ahk-install.exe

<!-- more -->

## 1. 先安装AutoHotKey

## 2. 打开记事本，把如下内容复制粘贴进去：

```Swift
; Typora
; 快捷增加字体颜色
; SendInput {Text} 解决中文输入法问题
#IfWinActive ahk_exe Typora.exe
{
    ; Ctrl+Alt+O 橙色
    ^!o::addFontColor("orange")
    ; Ctrl+Alt+R 红色
    ^!r::addFontColor("red")
    ; Ctrl+Alt+B 浅蓝色
    ^!b::addFontColor("cornflowerblue")
}
; 快捷增加字体颜色
addFontColor(color){
    clipboard := "" ; 清空剪切板
    Send {ctrl down}c{ctrl up} ; 复制
    SendInput {TEXT}<font color='%color%'>
    SendInput {ctrl down}v{ctrl up} ; 粘贴
    If(clipboard = ""){
        SendInput {TEXT}</font> ; Typora 在这不会自动补充
    }else{
        SendInput {TEXT}</ ; Typora中自动补全标签
    }

}
```

## 3. 将文件保存为ahk后缀的文件

如TyporaHotKey.ahk

## 4. 双击运行

## 5. 在Typora软件里就可以使用快捷键：

如按`Ctrl+Alt+O`添加橙色，Ctrl+Alt+R 红色，按`Ctrl+\`取消样式！

也可以右键 `MyHotkeyScript.ahk` 脚本文件，点击`Compile Script`编译脚本成`exe`程序，就可以不用下载`Autohotkey`在其他电脑上运行了。

上面脚本只写了橙色、红色、浅蓝三种颜色，你可以按需照例增加其他颜色或快捷方式！

## 6. 加入到开机自启动

右键TyporaHotKey.ahk创建快捷方式

复制快捷方式到Windows7 此目录下：

`C:\Users\用户名\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\startup`

ok！