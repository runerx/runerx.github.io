---
title: VIM的开发配置
categories: 其他
tags: 
  - Linux
---

前段时间使用vscode进行远程开发，vscode的远程连接有时候莫名其妙连不上，切换成VIM试试。

<!-- more -->



### 基础配置

使用现成的配置：https://github.com/amix/vimrc

```
git clone --depth=1 https://github.com/amix/vimrc.git ~/.vim_runtime
sh ~/.vim_runtime/install_awesome_vimrc.sh
```

配置好后修改

 vim ~/.vim_runtime/vimrcs/basic.vim

使用"注释掉如下，该配置会将vim的`0`移动到行首改为非空的行首

```
 map 0 ^
```



### Vundle 插件管理

安装插件管理器Vundle

https://github.com/VundleVim/Vundle.vim

```
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

将如下配置放入到.vimrc顶部

```
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" The following are examples of different formats supported.
" Keep Plugin commands between vundle#begin/end.
" plugin on GitHub repo
Plugin 'tpope/vim-fugitive'
" plugin from http://vim-scripts.org/vim/scripts.html
" Plugin 'L9'
" Git plugin not hosted on GitHub
Plugin 'git://git.wincent.com/command-t.git'
" git repos on your local machine (i.e. when working on your own plugin)
Plugin 'file:///home/gmarik/path/to/plugin'
" The sparkup vim script is in a subdirectory of this repo called vim.
" Pass the path to set the runtimepath properly.
Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
" Install L9 and avoid a Naming conflict if you've already installed a
" different version somewhere else.
" Plugin 'ascenator/L9', {'name': 'newL9'}

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
```

安装插件

Launch `vim` and run `:PluginInstall`

To install from command line: `vim +PluginInstall +qall`





###  TMUX配置

```
vim ~/.tmux.conf
```

```
 setw -g mode-keys vi

 # 绑定快捷键为r
 bind r source-file ~/.tmux.conf \; display-message "Config reloaded.."

 # 绑定Ctrl+hjkl键为面板上下左右调整边缘的快捷指令
 bind -r ^k resizep -U 10 # 绑定Ctrl+k为往↑调整面板边缘10个单元格
 bind -r ^j resizep -D 10 # 绑定Ctrl+j为往↓调整面板边缘10个单元格
 bind -r ^h resizep -L 10 # 绑定Ctrl+h为往←调整面板边缘10个单元格
 bind -r ^l resizep -R 10 # 绑定Ctrl+l为往→调整面板边缘10个单元格



 # 绑定hjkl键为面板切换的上下左右键
 bind -r k select-pane -U # 绑定k为↑
 bind -r j select-pane -D # 绑定j为↓
 bind -r h select-pane -L # 绑定h为←
 bind -r l select-pane -R # 绑定l为→


 #鼠标支持
 #set-option -g mouse on

 #colors
 set -g default-terminal "xterm-256color"

 #关闭自动rename窗口
 setw -g automatic-rename off
 setw -g allow-rename off

```





###  Nerdtree

```
安装命令：
git clone https://github.com/scrooloose/nerdtree.git ~/.vim/bundle/nerdtree

重启Vim，在命令模式下输入NERDTree即可开启目录展示，默认是当前路径。
:NERDTree
```





### ctags



| 常用命令     | 作用                                       |
| :----------- | :----------------------------------------- |
| Ctrl + ]     | 跳转到光标所在变量、宏、函数的定义处       |
| Ctrl + T     | 返回到跳转前的位置                         |
| Ctrl + W + ] | 分割当前窗口，并在新窗口中显示跳转到的定义 |
| Ctrl + O     | 返回之前的位置                             |
| :ts          | 列出所有匹配的标签                         |



### go插件

```
Plugin 'fatih/vim-go'
```







### 参考

https://blog.easwy.com/archives/advanced-vim-skills-catalog/

https://zhuanlan.zhihu.com/p/85040099



