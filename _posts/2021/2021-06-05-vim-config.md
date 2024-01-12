---
layout:       post
title:        "vim折腾笔记"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Vim
---

作为一名Golang开发者，使用过各种编辑器/IDE，其中包括：

- GoLand：做过Java的应该对jetbrains的IDE有深深的情感。GoLand有JetBrains家IDE的所有特色，它不使用gopls而是用自己的language server，因此很快速，并且有着一套强大的工具集，调试代码非常方便。唯一的问题就是它不是免费的。
- VSCode：目前最好用的开源编辑器之一，近几年有赶超SublimeText的趋势。插件库庞大，可以用插件堆砌成一个IDE，目前我同事最多的选择。
- Emacs：老牌编辑器。因为不习惯Emcas-mode，所以我用的是Spacemacs的vim-mode。是我在用vim之前用的最多的编辑器。

比起传统的编辑模式，vim-mode的效率是要高上不少的，因此上面的所有编辑器我无一例外都会使用vim插件。但是比起GoLand和VSCode，对于我这种喜欢在终端操作的开发者，还是更喜欢vim，emcas这样的编辑器。它们可以让我快速在终端输入命令，并且可以随时快速地打开一些项目。另外，它们可以在一些无GUI的环境（例如服务器）运行。

其中，我很喜欢emacs中Spacemacs的快捷键模式，使用空格(SPC)作为leader键，快捷键采用容易记忆的简写方式配置，例如查看Git Blame使用SPC g b快捷键。我将这种快捷键配置习惯加到了vim中。

我放弃Spacemacs的原因是，它的加载速度太慢了，每次启动需要加载200多个layers，并且我不是很习惯emacs-lisp，相比于它，我认为vim-script要人性化太多。

因此我最终选择了纯vim作为我的生产工具。下面是最终效果图：

![vim-0](https://pic1.zhimg.com/80/v2-7be3ed56fd2067831d75fe2c1cd63060_720w.webp)

![vim-1](https://pic4.zhimg.com/80/v2-424fb4cb3f490b8530ed298390cedbe7_720w.webp)

自从vim8引入异步机制之后，速度还是很不错的，所以我没有去用Neovim。如果你用的是vim8以下的版本，那么更应该去使用nvim而不是vim。

以下插件的快捷键列表：

![keymap](https://pic4.zhimg.com/80/v2-749e061041a3361c4af71ec096454d03_720w.webp)

以及一些配置的其它基础快捷键：

| 操作                               | 快捷键              |
| ---------------------------------- | ------------------- |
| 打开内置终端                       | SPC '               |
| 在Visual模式下复制内容到系统剪切板 | SPC y y             |
| 在Visual模式下剪切内容到系统剪切板 | SPC x x             |
| 快速呼出帮助文档                   | SPC h               |
| 垂直分屏                           | SPC w /             |
| 水平分屏                           | SPC w -             |
| 调整垂直分屏尺寸                   | SPC w ]<br/>SPC w [ |

所有插件的详细配置会在下面给出。

## 基础配置

以下是一些基本的配置：

```vim
" 关闭兼容模式
set nocompatible

" 让退格符在插入模式下能够删除前一个字符，就像一般的编辑器那样
set backspace=indent,eol,start

" utf-8编码，在一些插件下要求这个配置
set encoding=UTF-8

" 显示相对行号
set number relativenumber

" 突出显示当前行
set cursorline

" 突出显示搜索匹配项
set showmatch

" tab相关配置 - tab占用4spaces
set ts=4
set shiftwidth=4
" 自动对齐
set autoindent

" vs分屏时默认在右边打开
set splitright

" 实时搜索，不必等按下<Enter>再进行搜索
set incsearch

" 搜索忽略大小写
set ignorecase

" 不允许生成swp文件，这些文件用于异常中断时的恢复
" 如果需要这个功能，注释掉这个配置即可
set noswapfile

" 启动语法高亮
syntax enable

" 命令行的高度高一些
set cmdheight=2

" vim自带的命令行补全
set wildmenu

" Ctrl-A 跳转到当前行首，就像Emacs那样
nnoremap <C-a> ^

" :w命令时常会误输入为:W，因此这里做一个映射
cnoreabbrev W w

" 部分文件使用marker折叠，方便快速定位
autocmd FileType vim set foldmethod=marker
autocmd FileType proto set foldmethod=marker
```

## 快捷键基础配置

以下快捷键不依赖任何插件，所以我单独拿出来进行配置：

```vim
" SPC(Space)作为Leader，就像Spacemacs默认那样
let mapleader=" "

" 打开内置终端
nnoremap <leader>' :vert ter<CR>

" 将在Visual Mode下选中的内容复制到系统剪切板
vmap <leader>yy "+yy
vmap <leader>xx "+xx

" 帮助文档
nnoremap <leader>h :vert help 

" 窗口控制快捷键
" 垂直分屏
nnoremap <leader>w/ :vs<CR>
" 水平分屏
nnoremap <leader>w- :sv<CR>
" 调整垂直分屏尺寸
nnoremap <leader>w[ :vertical resize+3<CR>
nnoremap <leader>w] :vertical resize-3<CR>
```

## vim-plug安装

[vim-plug](https://github.com/junegunn/vim-plug)是一个高效的vim插件管理器。vim-plug很轻量，迅速。它能异步加载插件，因此即使插件很多，vim的速度不会受到很大的影响。

使用以下命令安装：

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

在Linux下，可能因为DNS的原因导致`raw.githubusercontent.com`这个域名访问不了。可以直接将文件复制下来放到`~/.vim/autoload/plug.vim`即可。

## 插件配置

安装好vim-plug之后，将所有插件列表加到配置中：

```vim
call plug#begin('~/.vim/plugged')

  " 搜索插件
  Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }
  Plug 'junegunn/fzf.vim'

  " 文件树
  Plug 'scrooloose/nerdtree'
  Plug 'Xuyuanp/nerdtree-git-plugin'
  Plug 'jistr/vim-nerdtree-tabs'

  " 状态栏
  Plug 'vim-airline/vim-airline'

  " 编辑插件
  Plug 'jiangmiao/auto-pairs'
  Plug 'majutsushi/tagbar'
  Plug 'junegunn/vim-easy-align'

  " 快速注释
  Plug 'scrooloose/nerdcommenter'

  " 代码补全
  Plug 'neoclide/coc.nvim', {'branch': 'release'}

  " Git
  Plug 'tpope/vim-fugitive'
 
  " 主题
  Plug 'fioncat/vim-oceanicnext'

  " 快速清除buffer
  Plug 'fioncat/vim-bufclean'

  " vim-go的极简版，去除了gopls，以及所有coc拥有的功能
  Plug 'fioncat/vim-minigo'

  " 在sign-line显示marks
  Plug 'kshenoy/vim-signature'

call plug#end()
```

重启vim，输入`:PlugInstall`即可进行安装。

针对不同的插件有一些不同的个性化配置，需要在插件列表的后面加上（建议在把插件安装好之后再加入下面的配置，否则vim可能会报错）

## oceanicnext主题

使用以下配置设置vim主题：

```vim
" 设置VIM主题
colorscheme OceanicNext
set background=dark
set termguicolors
```

## [Airline](https://github.com/vim-airline/vim-airline)

一个状态栏插件，可以和coc.nvim联动实现在状态栏实时展示代码的error和warning数量。

oceanicnext主题里面自带了airline插件，因此vim启动时airline会自动使用oceanicnext的主题。

```vim
" 显示状态栏
set laststatus=2

" 开启上方的tabline功能
let g:airline#extensions#tabline#enabled = 1
" 只显示buffer编号，不显示bufno
let g:airline#extensions#tabline#buffer_nr_show = 0
let g:airline#extensions#tabline#buffer_idx_mode = 1

" 图标替换，这样底层状态栏error/warning那里可以好看一些
let g:airline#extensions#coc#error_symbol = '✗ '
let g:airline#extensions#coc#warning_symbol = '⚡ '

nnoremap <leader>bn :bn<CR>
nnoremap <leader>bp :bp<CR>
```

## [NERDTree](https://github.com/preservim/nerdtree)

一个vim必装的文件树插件。配置如下：

```vim
let g:NERDSpaceDelims=1
let g:NERDTreeMinimalUI=1

" 打开文件树
nnoremap <leader>tt :NERDTreeToggle<CR>
" 在文件树打开当前文件
nnoremap <leader>ff :NERDTreeFind<CR>
```

我没有用文件icons，如果有需要可以安装[vim-devicons](https://github.com/ryanoasis/vim-devicons)插件。

## [tagbar](https://github.com/preservim/tagbar)

这个插件可以展示当前代码的整体结构，并进行快速跳转。需要ctags的支持：

```bash
brew install ctags
```

配置如下：

```vim
let g:tagbar_type_go = {
    \ 'ctagstype' : 'go',
    \ 'kinds'     : [
        \ 'p:package',
        \ 'i:imports:1',
        \ 'c:constants',
        \ 'v:variables',
        \ 't:types',
        \ 'n:interfaces',
        \ 'w:fields',
        \ 'e:embedded',
        \ 'm:methods',
        \ 'r:constructor',
        \ 'f:functions'
    \ ],
    \ 'sro' : '.',
    \ 'kind2scope' : {
        \ 't' : 'ctype',
        \ 'n' : 'ntype'
    \ },
    \ 'scope2kind' : {
        \ 'ctype' : 't',
        \ 'ntype' : 'n'
    \ },
    \ 'ctagsbin'  : 'gotags',
    \ 'ctagsargs' : '-sort -silent'
\ }
" 打开tagbar
nnoremap <leader>tb :TagbarToggle<CR>
```

## [coc.nvim](https://github.com/neoclide/coc.nvim)

一个比较新的补全插件，我曾经也是使用YouCompleteMe的，但是coc的功能和使用实在比ycm好很多。coc有自己的插件库，采用类似VSCode的插件管理方式，在独立的json文件进行配置，并使用`:CocInstall`单独安装插件。

coc需要node.js的支持，可以到官网下载对应系统的安装包进行安装，安装后输入以下命令查看版本：

```bash
node -v
```

coc需要10.12以上的node版本。

在`.vimrc`中，coc的配置如下（以下配置来源于官方文档，删除了我不用的一些功能）：

```vim
" 代码补全样式，详见:help completeopt
set completeopt=menu,menuone

set hidden

" 如果支持，将diagnostic signs放到原生行号中，这样就不必再显示sign
" colume以节约空间
if has("nvim-0.5.0") || has("patch-8.1.1564")
  set signcolumn=number
else
  set signcolumn=yes
endif

" 代码补全时使用TAB和s-TAB进行快速补全
" 这个行为和ycm的默认行为一样
inoremap <silent><expr> <TAB>
      \ pumvisible() ? "\<C-n>" :
      \ <SID>check_back_space() ? "\<TAB>" :
      \ coc#refresh()
inoremap <expr><S-TAB> pumvisible() ? "\<C-p>" : "\<C-h>"
function! s:check_back_space() abort
  let col = col('.') - 1
  return !col || getline('.')[col - 1]  =~# '\s'
endfunction

" <Enter> 选择补全项
inoremap <silent><expr> <cr> pumvisible() ? coc#_select_confirm()
                              \: "\<C-g>u\<CR>\<c-r>=coc#on_enter()\<CR>"

function! s:show_documentation()
  if (index(['vim','help'], &filetype) >= 0)
    execute 'h '.expand('<cword>')
  elseif (coc#rpc#ready())
    call CocActionAsync('doHover')
  else
    execute '!' . &keywordprg . " " . expand('<cword>')
  endif
endfunction

" 高亮展示光标悬浮的引用
autocmd CursorHold * silent call CocActionAsync('highlight')

" GD - 代码跳转
nmap <silent> gd <Plug>(coc-definition)
" GR - 显示变量引用
nmap <silent> gr <Plug>(coc-references)
" K：展示文档
nnoremap <silent> K :call <SID>show_documentation()<CR>
" 变量改名
nmap <leader>rn <Plug>(coc-rename)
" 将选中的代码格式化
xmap <leader>f  <Plug>(coc-format-selected)
nmap <leader>f  <Plug>(coc-format-selected)
" 上一个/下一个错误
nnoremap <leader>en :call CocAction('diagnosticNext')<CR>
nnoremap <leader>ep :call CocAction('diagnosticPrevious')<CR>
" 打开错误列表
nnoremap <leader>ee :CocList diagnostics<CR>
" 代码折叠
nnoremap <leader>fd :call CocAction('fold')<CR>
```

除此之外，coc还需要单独配置，输入`:CocConfig`打开配置文件，输入：

```json
{
	"diagnostic.warningSign": ">>",
	"diagnostic.refreshOnInsertMode": true,
	"diagnostic.virtualText": true,
	"languageserver": {
 		"go": {
			"command": "gopls",
			"rootPatterns": ["go.mod"],
			"trace.server": "verbose",
			"filetypes": ["go"]
		}
  	}
}
```

因为日常开发还用到了Python，所以还需要安装Python的插件：`:CocInstall coc-python`。

coc内部使用LSP作为languageserver，所以不同语言都需要安装自己的LSP工具：

Go语言(gopls)：

```bash
go install golang.org/x/tools/gopls@latest
```

python语言(jedi)：

```bash
pip3 install jedi
```

其它语言需要自行参照文档。

## [mini-go](https://github.com/fioncat/vim-minigo)

本来我是直接用[vim-go](https://github.com/fatih/vim-go)。后来发现这个插件和coc一起使用时存在一些问题：

- vim-go和coc都会启动自己的gopls，导致有多个gopls同时运行，vim资源占用高。
- vim-go有一些功能，例如代码检查，跳转等在coc中已经提供了，二者一起使用时显得有点多余。

因此我基于原来的vim-go进行了很多删改，删去了大量gopls相关脚本，仅保留了一些coc缺失的功能，例如`GoAddTag`，`GoFillStruct`等。以及Go语言语法高亮。

配置如下：

```vim
" Go语法高亮
let g:go_highlight_types = 1
let g:go_highlight_fields = 1
let g:go_highlight_functions = 1
let g:go_highlight_function_calls = 1
let g:go_highlight_operators = 1
let g:go_highlight_extra_types = 1
let g:go_highlight_methods = 1
let g:go_highlight_generate_tags = 1

" 在:w时自动进行GoImports
function! GoReformat()
	call go#fmt#Format(1)
endfunction
autocmd BufWriteCmd *.go call GoReformat()

" 一些Go Tools
nnoremap gat :GoAddTags<CR>
nnoremap grt :GoRemoveTags<CR>
nnoremap gi  :GoImports<CR>
nnoremap gfs :GoFillStruct<CR>
```

Go tools需要单独安装一些二进制程序：

```bash
go install github.com/klauspost/asmfmt/cmd/asmfmt@latest
go install github.com/go-delve/delve/cmd/dlv@latest
go install github.com/kisielk/errcheck@latest
go install github.com/davidrjenni/reftools/cmd/fillstruct@latest
go install github.com/rogpeppe/godef@latest
go install golang.org/x/tools/cmd/goimports@latest
go install golang.org/x/lint/golint@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install github.com/fatih/gomodifytags@latest
go install golang.org/x/tools/cmd/gorename@latest
go install github.com/jstemmer/gotags@latest
go install golang.org/x/tools/cmd/guru@latest
go install github.com/josharian/impl@latest
go install honnef.co/go/tools/cmd/keyify@latest
go install github.com/fatih/motion@latest
go install github.com/koron/iferr@latest
```

## [fzf](https://github.com/junegunn/fzf.vim)

fzf是一个强大的搜索插件，可以替代`ctrlp`使用。并且支持各种维度的搜索。

fzf本身是一个用Go写的命令行程序，官方做了一个vim插件以方便在vim内部执行各种搜索命令。在mac/Linux下可以先单独安装fzf命令：

```bash
brew install fzf
```

可以让fzf命令配合vim以实现快速搜索并打开文件：

```bash
alias fvim='vim $(fzf --bind=tab:preview-down,btab:preview-up)'
```

将这个语句加到`shell profile`（例如`.bashrc`，`.zshrc`）中，可以使用fvim实现搜索并用vim打开文件。

在vim中，对fzf进行如下配置：

```vim
" 全屏展示搜索
let g:fzf_layout = { 'down': '~100%' }
let g:fzf_preview_window = ['down:50%']

" fzf搜索框colors配置，让其符合当前主题
let g:fzf_colors =
\ { 'fg':      ['fg', 'Normal'],
  \ 'bg':      ['bg', 'Normal'],
  \ 'hl':      ['fg', 'Function'],
  \ 'fg+':     ['fg', 'CursorLine', 'CursorColumn', 'Normal'],
  \ 'bg+':     ['bg', 'Visual', 'CursorColumn'],
  \ 'hl+':     ['fg', 'Statement'],
  \ 'info':    ['fg', 'PreProc'],
  \ 'border':  ['fg', 'Ignore'],
  \ 'prompt':  ['fg', 'Conditional'],
  \ 'pointer': ['fg', 'Exception'],
  \ 'marker':  ['fg', 'Keyword'],
  \ 'spinner': ['fg', 'Label'],
  \ 'header':  ['fg', 'Comment'] }

" RG搜索, filename表示对什么文件进行搜索
function! RGSearch(filename)
	let command_fmt = 'rg --with-filename --column --line-number' 
		\ . ' --no-heading --color=always --smart-case -- %s ' 
		\ . a:filename . ' || true'
    let initial_command = printf(command_fmt, shellescape(''))
    let reload_command = printf(command_fmt, '{q}')
    let spec = {'options': ['--phony', '--query', '', '--bind', 
		\ 'change:reload:'.reload_command]}
    call fzf#vim#grep(initial_command, 1, fzf#vim#with_preview(spec), 0)
endfunction

" 搜索文件
nnoremap <leader>sf :Files<CR>
" 搜索全局内容
nnoremap <leader>sg :call RGSearch('')<CR>
" 搜索当前文件内容
nnoremap <leader>sl :call RGSearch(fnameescape(expand('%')))<CR>
" 搜索buffer
nnoremap <leader>sb :Buffers<CR>
```

内容搜索需要用到rg命令，要单独安装：

```bash
brew install rg
```

## [fugitive](https://github.com/tpope/vim-fugitive)

这个插件提供了很多Git命令，但是我更喜欢在终端中操作git，因此这个插件我唯一用到的就是GitBlame：

```vim
nnoremap <leader>gb :Git blame<CR>
```

## [bufclean](https://github.com/fioncat/vim-bufclean)

这个插件可以一键删除那些没有在窗口打开的buffer，以高效化buffer的管理。

```vim
nnoremap <leader>bc :BufClean<CR>
```