---
extends: default.liquid
title: Vim as a Python IDE
date: 2013-02-08T21:03:50Z
tags:
---

I've been spending quite a bit of time with our good old buddy Python recently, and when I do, I always invite along our mutual friend Vim. He's a barrell of laughs and always knows of a quicker way to do things. So I've been getting the two acquainted more and more and Vim's turned into a totally different person. So I am going to share with you how I setup Vim as my tool of choice when working with Python code. It's by no means the definitive way of working, but it works for me. I know there's quite a few posts similar to this one, but these are the tools **I** find useful and use. If you have some suggestions, comments, or know of additional tools that might be useful, I would like to hear about them in the comments below.

If you are a Vim user you might find this useful. If you are new to Vim I suggest spending some time with it before installing any plugins. That way you get used to how Vim works out of the box. Once you're comfortable with using it, and you've gotten to grips with the Vim-way, go ahead and install some extras.

Right, let's get started.

First and foremost you should install [Vundle](https://github.com/gmarik/vundle). Vundle will make installing and updating all the other tools much easier. It's basically pathogen with a lot of nice extras, like installing the bundles itself from their Github repos (and other sources). For more info see the README in the Github repo. To install follow the instructions from the repo, which are repeated below:

```sourceCode
git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle
```

then add the following to your `` `.vimrc ``\`

```sourceCode
set nocompatible
filetype off

set rtp+=~/.vim/bundle/vundle/
call vundle#rc()

" let Vundle manage Vundle
" required!
Bundle 'gmarik/vundle'

" The bundles you install will be listed here

filetype plugin indent on

" The rest of your config follows here
```

Now if you launch Vim and run the command

```sourceCode
:BundleList
```

a new split should appear listing the Vundle bundle. Everything went ok in that case.

As you go through this article you can install each item separately or you can add the bundles to your `` `.vimrc ``\` one by one and then just install them all at once at the end if you prefer.

Ok, so having done that let's configure a few more things.

## Highlight excess line length

You'll probably want to set a restriction to line width for python files. I like to set this to 120 chars. 80 chars is usually the standard, but with modern displays we can allow ourselves a few more, but you are free to adjust as you like. To enable this excess highlighting, add the following lines to your `` `.vimrc ``\`

```sourceCode
augroup vimrc_autocmds
    autocmd!
    " highlight characters past column 120
    autocmd FileType python highlight Excess ctermbg=DarkGrey guibg=Black
    autocmd FileType python match Excess /\%120v.*/
    autocmd FileType python set nowrap
    augroup END
```

Anything that exceeds the line length will be highlighted black, feel free to change this colour to suit your colourscheme. It also turns off line wrapping for python files.

## Powerline

Next up we install [Powerline](https://github.com/Lokaltog/powerline) which looks like this:

[<img src="/images/powerline.png" alt="image" width="500" />](/images/powerline.png)

It shows you your current mode (NORMAL), the current branch in Git, the file you are editing and some other useful information.

Simply add:

```sourceCode
Bundle 'Lokaltog/powerline', {'rtp': 'powerline/bindings/vim/'}
```

to your `.vimrc` below the comment we added earlier `" The bundles you install will be listed here`. Restart Vim and run `:BundleList` again. Now you should also see the Powerline bundle listed there. To install it run the command `vim :BundleInstall`. You'll see Vundle process the list and report on the status of the installation. Hopefully everything went ok. Powerline however does require a few more things, most notably patched fonts to display the special characters it uses. You can get pre-patched fonts from [the powerline-fonts repo](https://github.com/Lokaltog/powerline-fonts). If your font isn't listed then the powerline repo does provide a font-patcher you can use to try and patch your font. How this is done however is outside the scope of this article. To select your font and ensure that Powerline is always shown, you will also need to add these two lines to your `.vimrc`

```
" Powerline setup
set guifont=DejaVu\ Sans\ Mono\ for\ Powerline\ 9
set laststatus=2
```

`laststatus`ensures that Powerline shows up even if you don't have any splits. Restart vim and hopefully you'll see your powerline appear at the bottom of you window. Please note that this version of Powerline is a Python based version and thus requires your Vim to be built with Python enabled. To check if it is run:

```
$> vim --version | grep -i python
```

from the commandline. If you see `+python` then you are ok. There is [another Powerline](https://github.com/Lokaltog/vim-powerline) that is a native Vim plugin should you not have Python enabled or prefer to use it over the Python version.

## Fugitive

[Fugitive](https://github.com/tpope/vim-fugitive) is a [Git](https://git-scm.com/) plugin. It basically wraps most Git commands so that you can call them from inside Vim. They are prefixed with `G`, for example `Gcommit` For example it allows you to stage files directly from Vim and make the commit. It also leverages VimDiff to perform conflict resolution, blame and the like. There's a whole set of screencasts on how to use it available from [Vim Casts](https://vimcasts.org/episodes/fugitive-vim---a-complement-to-command-line-git/) which I recommend watching. To install Fugitive, add its bundle to Vundle: `Bundle 'tpope/vim-fugitive'` Run `:BundleInstall` again to install it.

## NerdTree

[NerdTree](https://github.com/scrooloose/nerdtree) is a filebrowser that pops up in a split when you need it and features a tree like file browser (hence the _tree_ part in the name). It looks somewhat like this:

![image](/images/nerdtree.png)

As usual you just need to add its package to Vundle:

```sourceCode
Bundle 'scrooloose/nerdtree'
```

and `:BundleInstall` once you restart Vim. To activate with `F2` add the following to `.vimrc`:

```sourceCode
map <F2> :NERDTreeToggle<CR>
```

Press `F2` in vim and it will take you to the current working directory. Press `?` to see NerdTree's list of commands.

## Python mode

This is the big one. It basically adds all the Python functionality you could ever want in Vim. Things like Lint, codecompletion, documentation lookup, jump to classes, refactoring tools etc. You'll find it in [Python-mode](https://github.com/klen/python-mode)

Its bundle is:

```sourceCode
Bundle 'klen/python-mode'
```

Again, `:BundleInstall` to install it and then we probably want to configure some items. There's a lot to configure, so if you want the complete picture I suggest you head over to the [Github repo](https://github.com/klen/python-mode) and read the more complete docs, or run `:help python-mode` from inside Vim.

I found the following settings most useful personally, but you might want to tweak some settings to suit your needs and workflow. The following a copy-paste from my .vimrc. The keyboard shortcuts in the comments are the ones I find most useful and I keep them there for reference:

```sourceCode
" Python-mode
" Activate rope
" Keys:
" K             Show python docs
" <Ctrl-Space>  Rope autocomplete
" <Ctrl-c>g     Rope goto definition
" <Ctrl-c>d     Rope show documentation
" <Ctrl-c>f     Rope find occurrences
" <Leader>b     Set, unset breakpoint (g:pymode_breakpoint enabled)
" [[            Jump on previous class or function (normal, visual, operator modes)
" ]]            Jump on next class or function (normal, visual, operator modes)
" [M            Jump on previous class or method (normal, visual, operator modes)
" ]M            Jump on next class or method (normal, visual, operator modes)
let g:pymode_rope = 1

" Documentation
let g:pymode_doc = 1
let g:pymode_doc_key = 'K'

"Linting
let g:pymode_lint = 1
let g:pymode_lint_checker = "pyflakes,pep8"
" Auto check on save
let g:pymode_lint_write = 1

" Support virtualenv
let g:pymode_virtualenv = 1

" Enable breakpoints plugin
let g:pymode_breakpoint = 1
let g:pymode_breakpoint_bind = '<leader>b'

" syntax highlighting
let g:pymode_syntax = 1
let g:pymode_syntax_all = 1
let g:pymode_syntax_indent_errors = g:pymode_syntax_all
let g:pymode_syntax_space_errors = g:pymode_syntax_all

" Don't autofold code
let g:pymode_folding = 0
```

To explain the above a bit, here's what it does:

- Allow me to look up Python docs by pressing `K`
- Automatically check my code on each save, but only use `PyLint` or `PyFlakes`
- Support virtualenv
- Use `<leader>b` to add a pdb shortcut (inserts `import pdb; pdb.set_trace() # XXX BREAKPOINT` into your code
- Enhanced syntax highlighting and formatting

As I said, please read the full docs and adjust the settings as you see fit.

## Jedi vim

Since I wrote this article I have discovered [Jedi-vim](https://github.com/davidhalter/jedi-vim) which I now use as the autocompletion tool instead of the rope plugin that comes with Python Mode. All you need to do is add the plugin to the vundle list and turn off Rope by replacing the `let g:pymode_rope = 1` with `let g:pymode_rope = 0`. I feel it's snappier and more capable than Rope. But if you want to avoid installing another plugin, then feel free to stay with Rope.

## Other settings

I also use some specific Vim settings in `.vimrc` that make the experience a bit nicer for me:

```sourceCode
" Use <leader>l to toggle display of whitespace
nmap <leader>l :set list!<CR>
" automatically change window's cwd to file's dir
set autochdir

" I'm prefer spaces to tabs
set tabstop=4
set shiftwidth=4
set expandtab

" more subtle popup colors
if has ('gui_running')
    highlight Pmenu guibg=#cccccc gui=bold
endif
```

## Summary

This is basically the crux of my Python and Vim development setup. I think the core of the whole thing really is python mode as it provides the most Python specific tools. The other plugins however do add some really useful functionality to make your life a little easier. You might ask why I don't list things like `fuzzy file search` and such, and that's because I don't use it. I've tried it before and didn't really get on with it very well and I prefer to either just open the files directly or using `NerdTree`.

I hope that this post provides some pointers to help you setup your Vim based Python development environment. As I said above, feel free to leave a comment with any plugins or settings that you find useful, always happy to hear about what else is out there.

Thanks for reading.
