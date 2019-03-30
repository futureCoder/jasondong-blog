---
title: "Linux系统下常用软件及配置"
date: 2017-12-26T23:26:51+08:00
hierarchicalCategories: true
draft: true
categories: 
- 分类
- 随手一写
---

# 终极的 Shell 配置 -- Oh My Zsh

## 首先安装zsh：
1. 安装zsh   
```sudo apt-get install zsh```
1. 检查zsh的版本，应高于4.3.9  
```zsh --version```
1. 将zsh作为默认shell  
```chsh -s $(which zsh)```
1. 注销重新登陆

## 安装curl或wget
```sudo apt-get install wget```
## 安装Git
```sudo apt-get install git```
## 使用curl或者wget安装Oh My Zsh
```sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"```
```sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"```
安装完成，可选择主题
```
ZSH_THEME="agnoster" # (this is one of the fancy ones)
# see https://github.com/robbyrussell/oh-my-zsh/wiki/Themes#agnoster
```
用这个主题需要安装字体 [Powerline fonts](https://github.com/powerline/fonts)  
```git clone https://github.com/powerline/fonts.git --depth=1```

## 准备源码阅读环境
### Vim基本设置
```
$ cat ~/.vimrc
set encoding=UTF-8
set langmenu=zh_CN.UTF-8
language message zh_CN.UTF-8
set fileencodings=ucs-bom,utf-8,cp936,gb18030,big5,euc-jp,euc-kr,latin1
set fileencoding=utf-8
sy on
set ai
set nu
set shiftwidth=2
set tabstop=4
set softtabstop=4
set ic "搜索时忽略大小写，以便在某些“骆驼”变量名风格的源码中查找
```
## 在Vim中使用Cscope
1. 安装Cscope
sudo apt-get install cscope
1. 确认Vim支持Cscope
vim --version | grep cscope
3. 给源代码建立索引
cscope-Rbq
其中，ctags 递归的在每个目录下生成 tags 文件，供 vim 读取;  
cscope 生成 cscope.out  
cscope.out: cscope reference data version 15 with inverted index  
将以下内容添加到 ~/.vimrc 中，以自动加载 cscope.out.  
```
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" cscope setting

if has("cscope")
   set csprg=/usr/bin/cscope              "指定用来执行 cscope 的命令
   set csto=1                             "先搜索tags标签文件，再搜索cscope数据库
   set cst                                "使用|:cstag|(:cs find g)，而不是缺省的:tag
   set nocsverb                           "不显示添加数据库是否成功
   " add any database in current directory
   if filereadable(ncscope.out")
      cs add cscope.out                   "添加cscope数据库
   endif
   set csverb                             "显示添加成功与否
endif

nmap <C-@>s :cs find s <C-R>=expand("<cword>")<CR><CR>  
nmap <C-@>g :cs find g <C-R>=expand("<cword>")<CR><CR>  
nmap <C-@>c :cs find c <C-R>=expand("<cword>")<CR><CR>  
nmap <C-@>t :cs find t <C-R>=expand("<cword>")<CR><CR>  
nmap <C-@>e :cs find e <C-R>=expand("<cword>")<CR><CR>  
nmap <C-@>f :cs find f <C-R>=expand("<cfile>")<CR><CR>  
nmap <C-@>i :cs find i <C-R>=expand("<cfile>")<CR><CR>  
nmap <C-@>d :cs find d <C-R>=expand("<cword>")<CR><CR>  
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
```
根据上面的 .vimrc，使用简单的组合键和字母即可使用 cscope 的查找功能。  
例如，<C-@>g 先按 ctrl+@，再按 g，即可查看当前光标所在符号的定义。  
:cs help  
s: 查找 C 语言符号，即查找函数名、宏、枚举值等出现的地方  
g: 查找函数、宏、枚举等定义的位置，类似 ctags 所提供的功能  
d: 查找本函数调用的函数  
c: 查找调用本函数的函数  
t: 查找指定的字符串  
e: 查找 egrep 模式，相当于 egrep 功能，但查找速度快多了  
f: 查找并打开文件，类似 vim 的 find 功能  
i: 查找包含本文件的文件  
## 在 vim 中使用 ctags
在源代码根目录下执行 ctags -R 命令用来为程序源代码生成标签文件，-R 选项表示递归操作，同时为子目录也生成标签文件。vim 利用生成的标签文件，可以进行相应检索、并在不同的文件中的C语言元素之间来回切换。  
需要在 tags 文件所在的目录下运行 vim。否则，要用 :set tags=xxx 指定 tags 文件的路径。  
使用 tag 命令时，可以使用 TAB 键进行匹配查找，继续按 TAB 键向下切换。  
跳到函数或数据结构xxx处  
:tag xxx  
跳到第一个定义处，优先跳转到当前文件  
:tnext  
跳到第一个  
:tfirst  
跳到前 count 个  
:[count]tprevious  
跳到后 count 个  
:[count]tnext  
跳到最后一个  
:tlast  
在所有 tagname 中选择  
:tselect tagname  
跳到包含 block 的标识符  
:tag /block   
(这里 '/' 就是告诉 vim，'block' 是一个语句块标签。)  

跳转到光标所在函数标识符的定义处  
Ctrl+]  
使用"ctrl+t"退回上层  
Ctrl+t  
在以 write_ 开头的标识符中选择  
:tselect /^write_  
这里，'^'表示开头，同理，'$'表示末尾。  
在函数中移动光标的快捷键:  

[{ 转到上一个位于第一列的"{"  
}] 转到下一个位于第一列的"{"  
{ 转到上一个空行  
} 转到下一个空行  
gd 转到当前光标所指的局部变量的定义  
\* 转到当前光标所指的单词下一次出现的地方  
\# 转到当前光标所指的单词上一次出现的地方  

## 使用 taglist 显示 symbol 窗口
taglist 插件可以像 Source Insight 那样将当前文件中的宏、全局变量、函数等 tag 显示在 Symbol 窗口，用鼠标点上述 tag，就跳到该 tag 定义的位置；可以按字母序、该tag所属的类或scope，以及该 tag 在文件中出现的位置进行排序；如果切换到另外一个文件，Symbol 窗口更新显示这个文件中的 tag。taglist 依赖于 ctags。  
打开 vim 的文件类型自动检测功能；  
系统中装了 Exuberant ctags 工具，并且 taglist 能够找到此工具（因为 taglist 需要调用它来生成 tag 文件）；  
vim 支持 system() 调用；  
在 ~/.vimrc 中加入  
```
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" ctags setting
set tags=./tags,./../tags,./*/tags;

" Tag list (ctags)

filetype on                            "文件类型自动检测

if MySys() == "windows"                "设定windows系统中ctags程序的位置
   let Tlist_Ctags_Cmd = 'ctags'
elseif MySys() == "linux"              "设定linux系统中ctags程序的位置
   let Tlist_Ctags_Cmd = '/usr/bin/ctags'
endif

let Tlist_Show_One_File = 1            "不同时显示多个文件的tag，只显示当前文件的
let Tlist_Exit_OnlyWindow = 1          "如果taglist窗口是最后一个窗口，则退出vim
let Tlist_Use_Right_Window = 1         "在右侧窗口中显示taglist窗口 
  
map <silent> <F8> :TlistToggle<cr>     "在映射F8键打开tags窗口
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
```

# 安装插件管理器Vundle
```git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim```
## 安装YCM(YouCompleteMe)
在Vundle中加入  
Plugin 'Valloric/YouCompleteMe'
## 引号/括号匹配
Plugin 'Raimondi/delimitMate'
## 代码格式化插件 Formatter
Plugin 'Chiel92/vim-autoformat'
let g:formatdef_harttle = '"astyle --style=attach --pad-oper"'
let g:formatters_cpp = ['harttle']
let g:formatters_java = ['harttle']