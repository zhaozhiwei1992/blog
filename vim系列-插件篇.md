---
title: vim系列-插件篇
date: 2024-08-29
updated: 2024-08-29
tags: [vim, 插件]
---

# Vim插件管理器Vundle使用指南

## Vundle简介

Vundle是一个Vim插件管理器，它允许您：
- 在`.vimrc`中跟踪和配置插件
- 安装配置的插件
- 更新插件
- 搜索所有可用的Vim脚本
- 清理未使用的插件
- 通过交互模式一键运行上述操作

## 快速开始

### 1. 安装Vundle

```shell
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

### 2. 配置`vimrc.bundle`

在`.vimrc`中添加以下内容，以使用Vundle：

```vim
... 一堆配置
if filereadable(expand("~/.vimrc.bundles"))
    source ~/.vimrc.bundles
endif
filetype plugin indent on

```

### 3. 安装插件

在Vim中执行`:PluginInstall`来安装插件。

或者从命令行执行`vim +PluginInstall +qall`。

### 4. （可选）对于使用fish shell的用户

在`.vimrc`中添加`set shell=/bin/bash`。

## 好用的Vim插件

### surround

surround插件可以方便地添加、修改和删除包围符：

```vim
" 替换: cs"'
"Hello world!" -> 'Hello world!'

" 替换-标签: cst"
<a>abc</a>  -> "abc"

" 添加: ysiw"
Hello -> "Hello"

" 添加-整行: yss"
Hello world -> "Hello world"

" 添加-当前到行尾: ys$"
```

### vim-repeat

vim-repeat插件允许你重复一个插件的操作，例如surround.vim：

通过`.`号重复上一次操作。

### 模糊搜索神器fzf

fzf是一个强大的模糊搜索工具，结合fzf.vim插件使用：

```vim
Bundle 'junegunn/fzf'
Bundle 'junegunn/fzf.vim'
```

在Vim中执行`:PlugInstall`安装插件。

需要安装`ripgrep`：

```shell
pacman -S ripgrep
```

## 使用Vundle

Vundle提供了多种命令来管理插件：

- `:PluginList` - 列出配置的插件
- `:PluginInstall` - 安装插件
- `:PluginUpdate` - 更新插件
- `:PluginSearch` - 搜索插件
- `:PluginClean` - 清理未使用的插件

```vim
" 把自定义的.vimrc.bundle文件引入到.vimrc中
source ~/.vimrc.bundle
```

```shell
# 复制molokai主题到你的vim目录
cd ~/.vim & cp -r bundle/molokai/colors .
```

```vim
" 更新所有插件
:BundleUpdate
```

### 一份个人插件配置
``` vim
if &compatible
  set nocompatible
end

filetype off
set rtp+=~/.vim/bundle/Vundle.vim/
call vundle#rc()

" Let Vundle manage Vundle
Bundle 'gmarik/vundle'

" Define bundles via Github repos
Bundle "tomasr/molokai"
Bundle 'vim-airline'
Bundle 'jlanzarotta/bufexplorer'
"Bundle 'skammer/vim-css-color'
"Bundle 'mattn/emmet-vim'
"Bundle 'mbbill/fencview'
"Bundle 'tpope/vim-fugitive'
Bundle 'tpope/vim-surround'
Bundle 'tpope/vim-repeat'
"Bundle 'airblade/vim-gitgutter'
"Bundle 'nathanaelkane/vim-indent-guides'
"Bundle 'vim-scripts/JavaScript-Libraries-Syntax'
"Bundle 'scrooloose/nerdcommenter'
Bundle 'scrooloose/nerdtree'
"Bundle 'edkolev/promptline.vim'
"Bundle 'MarcWeber/vim-addon-mw-utils'
"Bundle 'tomtom/tlib_vim'
"Bundle 'garbas/vim-snipmate'
Bundle 'scrooloose/syntastic'
"Bundle 'godlygeek/tabular'
Bundle 'majutsushi/tagbar'
"Bundle 'tomtom/tlib_vim'
"Bundle 'vim-scripts/TxtBrowser'
"Bundle 'xsbeats/vim-blade'
"Bundle 'guns/vim-clojure-static'
"Bundle 'kchmck/vim-coffee-script'
"Bundle 'webfd/vim-cppstl'
"Bundle 'rhysd/vim-crystal'
"Bundle 'hail2u/vim-css3-syntax'
"Bundle 'OrangeT/vim-csharp'
"Bundle 'chrisbra/csv.vim'
"Bundle 'JesseKPhillips/d.vim'
"Bundle 'dart-lang/dart-vim-plugin'
"Bundle 'ekalinin/Dockerfile.vim'
"Bundle 'elixir-lang/vim-elixir'
"Bundle 'jimenezrick/vimerl'
"Bundle 'kongo2002/fsharp-vim'
"Bundle 'fatih/vim-go'
"Bundle 'tfnico/vim-gradle'
"Bundle 'webfd/vim-haskell'
"Bundle 'jdonaldson/vaxe'
"Bundle 'othree/html5.vim'
"Bundle 'digitaltoad/vim-jade'
"Bundle 'pangloss/vim-javascript'
"Bundle 'mitsuhiko/vim-jinja'
"Bundle 'jason0x43/vim-js-indent'
"Bundle 'leshill/vim-json'
"Bundle 'elzr/vim-json'
"Bundle 'mxw/vim-jsx'
"Bundle 'briancollins/vim-jst'
"Bundle 'JuliaLang/julia-vim'
"Bundle 'udalov/kotlin-vim'
"Bundle 'groenewege/vim-less'
"Bundle 'godlygeek/tabular'
"Bundle 'plasticboy/vim-markdown'
"Bundle 'mustache/vim-mustache-handlebars'
"Bundle 'evanmiller/nginx-vim-syntax'
"Bundle 'rgrinberg/vim-ocaml'
"Bundle 'vim-perl/vim-perl'
"Bundle 'exu/pgsql.vim'
"Bundle 'shawncplus/phpcomplete.vim'
"Bundle 'stephpy/vim-php-cs-fixer'
"Bundle '2072/PHP-Indenting-for-VIm'
"Bundle 'me-vlad/python-syntax.vim'
"Bundle 'wlangstroth/vim-racket'
"Bundle 'vim-ruby/vim-ruby'
"Bundle 'rust-lang/rust.vim'
"Bundle 'derekwyatt/vim-scala'
"Bundle 'cakebaker/scss-syntax.vim'
"Bundle 'slim-template/vim-slim.git'
"Bundle 'iakio/smarty3.vim'
"Bundle 'keith/swift.vim'
"Bundle 'evidens/vim-twig'
"Bundle 'leafgarland/typescript-vim'
"Bundle 'tkztmk/vim-vala'
"Bundle 'vimwiki/vimwiki'
"Bundle 'xwsoul/vim-zephir'
Bundle 'junegunn/fzf'
Bundle 'junegunn/fzf.vim'

syntax enable
filetype plugin indent on

if filereadable(expand("~/.vimrc.bundles.local"))
  source ~/.vimrc.bundles.local
endif

filetype on
```
