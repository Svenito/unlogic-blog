---
extends: default.liquid
title: Vim settings done better
date: 2015-01-07T21:03:50Z
---

Today we'll cover a nice config setup for vim that I use. I got the idea from another blog post elsewhere and I am sorry to say I cannot locate the post right now. If you know of it, please leave a comment below and I will add it to the post as a credit.

I say 'better' rather than 'right' because I don't believe there's necessarily a right way to do it, only bad and better. Also, what works for one might not work for another, so please adjust as required.

The main principle for this setup is a modular rc file setup, where different files are responsible for different configurations. This way it's easier to find a specific setting rather than trawling through a long `.vimrc` file. So, without further delay, here's the main `.vimrc` (which we still need). The role of this file is to source all the other files we will create later. A [previous post](https://unlogic.co.uk/2013/02/08/vim-as-a-python-ide/) covers some configuration options specific for Python, if you are interested.

```sourceCode
let s:vim_home = '~/.vim/settings/'

let config_list = [
  \ 'plugins.vim',
  \ 'base.vim',
  \ 'functions.vim',
  \ 'theme.vim',
  \ 'settings.vim',
  \ 'leader.vim',
  \ 'keymappings.vim',
  \ 'languages.vim',
  \ 'plugin_settings.vim'
\]

for files in config_list
  for f in split(glob(s:vim_home.files), '\n')
    exec 'source '.f
  endfor

endfor
```

You can see how we'll divide up responsibility here. I won't go into details about the contents of each file, I will leave that up to you to decide. Basically here we set a directory - `~/.vim/settings/` - as the root for our setting files. Then we have a list of files which contain the configs:

- **plugins.vim**
  This contains my Vundle setup and required plugins. The reason this is first is because Vundle needs to have specific settings set that I change.

- **base.vim**
  Contains the basic config stuff like tab interpretation, indentation behaviour, and such.

- **functions.vim**
  Any vim functions you have written or use go in here

- **theme.vim**
  Theme configuration and selection

- **settings.vim**
  I don't have much in here, just setting my code folding prefs, basically anything that's not in base

- **leader.vim**
  Any leader customisations go in here

- **keymappings.vim**
  Like leader, but for keymaps

- **languages.vim**
  Any language specific settings

- **plugin_settings.vim**
  Configurations for all plugins. You might want to split this out further if it gets big

I've been using this for a while now and am very pleased with how it works and how easy it is now to find that specific setting that I need to tweak. No more searching through a long .vimrc file, just go directly to a much smaller file and get back to your code much quicker.

Hope you find this useful too. Feel free to share any other config tips in the comments below.
