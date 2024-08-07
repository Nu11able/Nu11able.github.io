---
title: vim
date: 2022-02-20
tags:
 - tools
categories:
 - tools
---

# vim使用记录
## `单按键命令`
|按键|功能|
|-|-|
|h|前|
|j|下|
|k|上|
|l|后|
|ESC|进入nomal模式|
|i|进入INSERT模式|
|v|进入选择模式|
|V|选择整行|
|y|复制|
|p|粘贴|
|0|将光标移到行首(数字0)|
|^|将光标移到该行第一个字母位置|
|$|将光标移到行位|
|a|在光标后插入|
|A|在行尾插入(相当于$a)|
|o|在下一行插入(相当于vscode中ctrl+enter)
|O|在上一行插入(相当于vscode中ctrl+shift+enter)
|x|删除一个字符|
|w|移动至下一个单词开始位置|
|e|移动至当前单词末尾|
|b|后退一个单词|
|u|撤销命令|
|U`？`|To undo all the changes on a line
|p|将粘贴板的内容复制到光标后面(lowercase)
|P|将粘贴板的内容复制到光标前面(uppercase)

## 窗口分割
|命令|功能|
|-|-|
|:sp [filename]|水平分割窗口|
|:vsplit [filename]|垂直分割窗口|
|Ctrl+w+方向键|切换到前／下／上／后一个窗格|
|Ctrl+w+h/j/k/l|同上|
|Ctrl+ww|依次向后切换到下一个窗格中|


## `组合按键命令`
|按键|功能|
|-|-|
|dw|删除至下一个单词开始|
|de|删除至当前单词末尾|
|dd|删除整行|
|d$|删除光标位置到行尾|
|r?|r+任意按键：将光标位置修改为所按按键
|ce|删除光标至当前单词末尾并进入INSERT模式(相当于dei)

- **d \[number\] motion**, d: delete
- **c \[number\] motion**, c: change 

## `命令说明`
|命令|功能|
|-|-|
|/word|搜索word|
|s/old/new|to substitute 'new' for 'old'(只替换一次)
|s/old/new/g|to substitute 'new' for 'old'(修改该行内所有匹配)
|#,#s/old/new/g|where #,# are the line numbers of the range of lines where the substitution is to be done.
|%s/old/new/g|to change every occurrence in the whole file.
|%s/old/new/gc|to find every occurrence in the whole file, with a prompt whether to substitute or not.(c : confirm)
|!command|execute the external command.


## `其他快捷按键`
- >Type  %  to find a matching ),], or }
    
    在[, (或 { 按下 % 可以快速移动到与之匹配的括号处

- 在选择模式下按下:进入命令模式w TEST会将选中的内容保存到TEST文件中

- 在命令模式下: r FILENAME 会将FILENAME的内容插入到下一行，**r !ls将ls命令的执行结果插入到后一行**

- To go back to where you came from press  **CTRL-O**  (Keep Ctrl down while pressing the letter o).  Repeat to go back further.  CTRL-I goes forward.