---
title: exvim的配置过程
layout: post
category: Linux
tag:
    - tool
    - vim
---
[Exvim](http://exvim.github.io/docs/intro/)是一个将vim集成为开发环境的vim的项目。
其非常方便的让你在管理自己的配置文件及vim插件。
个人觉得exvim的几个非常好的特色在于：

*   优秀的项目管理
*   高性能的全局搜索
*   良好的vim插件管理

当然还有一些其它的特色，请去项目主页查看。下面介绍自己配置exvim环境的过程：

###FORK原项目及安装

由于觉得在使用过程可能会有一些自己的修改化配置，甚至一些其它可能会涉及到插件的修改，
想要将这些变动持续到github中，因此从[原项目](https://github.com/exvim/main)fork了一份用于
自己exvim的开始。

首先，是将项目clone到本地

    mkdir Exvim
    git clone git@github:XXXX/main.git

克隆完成后，先不急于安装，因为通常情况下，会缺少一三个必要的tool(虽然缺少时不会影响安装的完成)，需要
先将这三个tool进行安装。这里仅说明[idutils](http://www.gnu.org/software/idutils/)的安装。

idutils的安装需要到项目主面下载源码进行安装。下载完成源码后进入目录.

    cd idutils-4.5/
    ./configure
    make
    make install

完成安装后，就可以进行exvim的安装了，步骤如下：

    cd Exvim/main
    sh unix/install.sh

到这里exvim项目算简单的安装完成。

###替换原vim

实际上些时已经通过运行

    sh unix/gvim.sh

体验exvim的效果了，不过对于linux，需要将`unix/gvim.sh`脚本中的`gvim`命令替换为`vim`
体验完成后觉得效果可以，可以使用`sh unix/replace-my-vim.sh` 完成将exvim替换现有vim配置的操作。
不过为了能够备份原vim的配置，及同时将github管理下的exvim能够与使用的vim配置保持关系，将
replace-my-vim.sh脚本修改成如下：

```sh
    #!/bin/bash

    ############################  BASIC SETUP TOOLS
    msg() {
        printf '%b\n' "$1" >&2
    }

    success() {
        if [ "$ret" -eq '0' ]; then
        msg "\e[32m[✔]\e[0m ${1}${2}"
        fi
    }

    ############################ SETUP FUNCTIONS
    lnif() {
        if [ -e "$1" ]; then
            ln -sf "$1" "$2"
        fi
        ret="$?"
    }

    do_backup() {
        smsg="$1"
        shift
        today=`date +%Y%m%d_%s`
        while [ $# -gt 0 ]; do
            [ -e "$1" ] && [ ! -L "$1" ] && mv "$1" "$1.$today";
            shift
        done
        ret="$?"
        success "$smsg"
    }

    create_symlinks() {
        endpath=`pwd`

        lnif "$endpath/dist/ctags_lang"         "$HOME/.ctags"
        lnif "$endpath/.vimrc"                  "$HOME/.vimrc"
        lnif "$endpath/.vimrc.local"            "$HOME/.vimrc.local"
        lnif "$endpath/.vimrc.plugins"          "$HOME/.vimrc.plugins"
        lnif "$endpath/.vimrc.plugins.local"    "$HOME/.vimrc.plugins.local"
        lnif "$endpath/.vimrc.mini"             "$HOME/.vimrc.mini"
        lnif "$endpath/vimfiles"                "$HOME/.vim"

        ret="$?"
        success "$1"
    }
    ############################ MAIN()
    do_backup   "Your old vim stuff has a suffix now and looks like .vim.`date +%Y%m%d%S`" \
                "$HOME/.vim" \
                "$HOME/.vimrc" \
                "$HOME/.vimrc.local" \
                "$HOME/.vimrc.plugins" \
                "$HOME/.vimrc.plugins.local" \
                "$HOME/.ctags"

    create_symlinks "Setting up vim symlinks"
```
这样老的vim配置进行了备份，同时exvim的配置也通过软连接的形式连接到了新的vim的配置中。

###exvim的使用
这部分就不多说了，见项目主页，比较重要的部分是对
`file-filter`选项的配置，这个选项会一定程度影响你执行`:Update`命令时建立tags,ID的文件范围，如果不包含
只会对默认的几类文件类型建立配置

###插件的修改及添加

####插件的修改
这部分主要是原有的配置下有几个部分是为影响使用体验的。  

*  ex-config: 这个部分主要是先中时目标行与选择行高亮的效果修改如下：

```sh
    cd .vim/bundle/ex-config/after
    vim solarized.vim
    "将else后的exConfirmLine修改为如下：
    hi clear exConfirmLine
    " hi exConfirmLine gui=none guibg=#ffe4b3 term=none cterm=none ctermbg=82
    hi exConfirmLine term=reverse cterm=reverse ctermfg=245 ctermbg=235 gui=bold,reverse

    hi clear exTargetLine
    " hi exTargetLine gui=none guibg=#ffe4b3 term=none cterm=none ctermbg=82
    hi exTargetLine term=reverse cterm=reverse ctermfg=245 ctermbg=235 gui=bold,reverse
```

*  ex-gsearch: 需要更改lid的路径为自己主机的上lid的路径

```sh
    if ignore_case
        echomsg 'search ' . a:pattern . '...(case insensitive)'
        let cmd = '/usr/local/bin/lid --result=grep -i -f"' . id_path . '" ' . a:method . ' ' . a:pattern
    else
        echomsg 'search ' . a:pattern . '...(case sensitive)'
        let cmd = '/usr/local/bin/lid --result=grep -f"' . id_path . '" ' . a:method . ' ' . a:pattern
    endif
```
    
####插件的添加
主要添加了三个插件，如下：

    Bundle 'Yggdroot/indentLine'
    Bundle 'spf13/PIV'
    Bundle 'jiangmiao/auto-pairs'

####插件配置的修改
主要是简单的配置修改，如下：

    " 常规模式下输入 F2 调用插件
    nmap <F2> :NERDTreeToggle<CR>
    " 设置Gvim的对齐线样式
    "if g:isGUI
    let g:indentLine_char = "|"
    let g:indentLine_first_char = "|"
    "endif

另還有一個因為PIV導致的php文件的不缩进问题，需要去掉PIV插件中的
    
    set noautoindent
    set nosmartindent

###.vimrc的修改
这部分主要是为了满足自己的使用习惯

    " ==================================================
    " 可用<C-k,j,h,l>切换到上下左右的窗口中去
    noremap <c-k> <c-w>k
    noremap <c-j> <c-w>j
    noremap <c-h> <c-w>h
    noremap <c-l> <c-w>l

    set smartindent                                       "启用智能对齐方式
    set foldenable                                        "启用折叠
    set smarttab                                          "指定按一次backspace就删除shiftwidth宽度的空格
    " 常规模式下输入 cS 清除行尾空格
    nmap cS :%s/\s\+$//g<cr>:noh<cr>

    " 常规模式下输入 cM 清除行尾 ^M 符号
    nmap cM :%s/\r$//g<cr>:noh<cr>
    " Ctrl + K 插入模式下光标向上移动
    imap <c-k> <Up>

    " Ctrl + J 插入模式下光标向下移动
    imap <c-j> <Down>

    " Ctrl + H 插入模式下光标向左移动
    imap <c-h> <Left>

    " Ctrl + L 插入模式下光标向右移动
    imap <c-l> <Right>
    set cmdheight=2                                       "设置命令行的高度为2，默认为1

好，后面的有空再加
