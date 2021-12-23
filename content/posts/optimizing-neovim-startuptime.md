---
title: "Optimizing Neovim Startuptime"
date: 2021-12-22T18:06:44-04:00
# weight: 1
aliases: ["/optimizing-neovim"]
tags: ["neovim", "lua"]
author: "NTBBloodbath"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: false
description: "This is a complete guide on how to optimize your Neovim configuration startuptime the hardcore way"
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
editPost:
    URL: "https://github.com/NTBBloodbath/ntbbloodbath.github.io/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

As you may know, if you bloat your Neovim configuration with a good amount of plugins it's going to get
slower and slower and this can be a VERY BIG problem because you will need to wait a lot of time to start
editing. This guide aims to help you resolve this slow startuptime issue.

Special thanks to [vhyrro](https://github.com/vhyrro), who taught me several tricks mentioned in this post :heart:.

> **Important**: Before starting, this guide is intended for Neovim >= 0.5 users and it's fully oriented to Lua.
> If you're using Vimscript I highly recommend you to leave that ugly monster alone and join the Lua side.

## Lazy-loading plugins

We will start from here, but first of all, what does lazy-loading means?

Lazy-loading can be read as "load on-demand". That means you load stuff once you need them. For example,
you will surely want to load autocompletion plugins once you start editing or enable a Rust plugin only when
editing Rust files.

For this specific task I recommend using [packer.nvim](https://github.com/wbthomason/packer.nvim). This is an
advanced plugins manager that allows you to lazy-load your plugins in an easy and declarative way.

### Installing packer.nvim

For installing `packer.nvim` we will also lazy-load it because why not?

Having said that, let's install it!

> **Important**: if you're already using `packer.nvim` and lazy-loading it you can safely skip this step.

The first thing that you will need to do is create a new file into your Neovim lua directory (`~/.config/nvim/lua`
on *nix systems). Call it as you want, I'll call it as `plugins.lua`.

Once you've done that, we can begin with `packer.nvim` installation. We aren't going to install it using the README
way because we're going to lazy-load it and do some extra small steps to customize bootstrapping.

> **Note**: bootstrapping means "load a set of instructions when X is launched or turned on".

Let's start with our `packer.nvim` bootstrapping so `packer.nvim` will be automatically downloaded if not found in your system.

```lua
-- /home/user/.local/share/nvim/site/pack/packer/opt/packer.nvim
local packer_path =
  vim.fn.stdpath("data") .. "/site/pack/packer/opt/packer.nvim"

if vim.fn.empty(vim.fn.glob(packer_path)) > 0 then
  vim.notify("Bootstrapping packer.nvim, please wait ...")
  vim.fn.system({
    "git",
    "clone",
    "https://github.com/wbthomason/packer.nvim",
    packer_path,
  })
end

vim.cmd("packadd packer.nvim")
local packer = require("packer")

packer.startup(function(use)
  -- Plugins manager
  use({
    "wbthomason/packer.nvim",
    opt = true,
  })
end)
```

And now, what does this code means? Let me explain it to you!

- In the first two lines we're defining what our `packer.nvim` installation path is.
- After those lines we have a conditional that checks if `packer.nvim` is installed or not and installs it if not installed.
- Then we manually load `packer.nvim` plugin after checking its existence in our system and require it.
- Finally we start `packer.nvim` with the `packer.startup` function where we declare our plugins and lazy-load `packer.nvim` itself.

Now just require that module in your `init.lua` and relaunch Neovim and `packer.nvim` will be automatically installed and loaded.

### How to lazy-load your plugins the proper way

Once you have `packer.nvim` installed and working we can continue with the next step, that is lazy-load your
plugins!

> **Important**: you should never lazy-load `nvim-lspconfig` if you don't want to have issues with EFM or diagnosticls language servers
> and also have faster lsp startup.

For example, we will lazy-load three big libraries that are used in some plugins and takes a ton of time.

_Note that we're omitting the initial `packer.nvim` setup that we did before when installing `packer.nvim`_.

```lua
use({
  "kyazdani42/nvim-web-devicons",
  module = "nvim-web-devicons",
})

use({
  "nvim-lua/plenary.nvim",
  module = "plenary",
})

use({
  "nvim-lua/popup.nvim",
  module = "popup",
})
```

Here we tell `packer.nvim` to load these plugins when we require their Lua modules with a `require` function
(e.g. `local plenary = require("plenary")`) so they will be loaded only when another plugin requires them.

We can also lazy-load plugins in a TON of different ways too. For example, we can load plugins when we trigger certain commands, events, keybinds, etc.
The following code chunk are some examples.

```lua
-- Neogit,
-- load only once we use `:Neogit` command
use({
  "TimUntersberger/neogit",
  config = function()
    require("neogit").setup({})
  end,
  cmd = "Neogit",
})


-- Tabline,
-- load when triggering BufWinEnter event
use({
  "akinsho/bufferline.nvim",
  config = function()
    -- Config goes here ...
  end,
  event = "BufWinEnter",
})

-- Autocomplete HTML tags,
-- load after loading treesitter plugin
use({
  "windwp/nvim-ts-autotag",
  after = "nvim-treesitter",
})
```

> You can find more information about the different ways to lazy-load plugins
> in [Specifying plugins](https://github.com/wbthomason/packer.nvim#specifying-plugins) section of packer's readme.

#### Manually lazy-load plugins with packer.nvim

We can also make use of `:PackerLoad` command to manually load plugins with `packer.nvim`. That command makes use of
built-in Neovim `:packadd` and `:source` commands under the hood.

For example, if we want to lazy-load `nvim-treesitter` plugin manually we could do the following.


```lua
----- In our plugins.lua -------
use({
  "nvim-treesitter/nvim-treesitter",
  opt = true,
  run = ":TSUpdate",
  config = function()
    -- Config goes here ...
  end,
})

----- In our init.lua ----------
-- This conditional ensures packer and treesitter plugin are
-- installed before trying to load treesitter plugin
if packer_plugins and packer_plugins["nvim-treesitter"] then
  vim.cmd("PackerLoad nvim-treesitter")
end
```

You can read further about how `PackerLoad` works internally [here](https://github.com/wbthomason/packer.nvim/blob/master/lua/packer/load.lua#L118-L159).

## Altering Neovim defaults

There are other ways to also improve Neovim startuptime and altering some Neovim defaults is one of those ways.
For example, you could temporarily disable syntax highlighting and ftplugin on launch to reduce your startuptime in around 20ms _or even more_
and defer some things.

### Temporarily disabling syntax highlighting and filetype

```lua
-- This needs to be at top of your `init.lua`
vim.cmd([[
  syntax off
  filetype off
  filetype plugin indent off
]])
```

After this, we will want to re-enable those options at the end of our `init.lua`.
We will use the same code snippet but using `on` instead of `off`.

```lua
-- This needs to be at bottom of your `init.lua`
vim.cmd([[
  syntax on
  filetype on
  filetype plugin indent on
]])
```

### Defer code chunks

When we defer chunks of code we are actually using a one-shot timer that calls a function
that executes some code. That means, defer calling X function until Y milliseconds passes.

#### How can we defer our code?

Neovim Lua API comes with several functions to schedule functions execution, like `schedule`.
However, there is a specific one that we're going to use that is called `defer_fn`.

_From `:h vim.defer_fn()`_:

```vim
vim.defer_fn({fn}, {timeout})                                    *vim.defer_fn*
    Defers calling {fn} until {timeout} ms passes.  Use to do a one-shot timer
    that calls {fn}.

    Note: The {fn} is |schedule_wrap|ped automatically, so API functions are
    safe to call.

    Parameters: ~
        {fn}        Callback to call once {timeout} expires
        {timeout}   Time in ms to wait before calling {fn}

    Returns: ~
        |vim.loop|.new_timer() object

```

As we can see, `vim.defer_fn` gets two non-optional parameters. A function and a timeout (in milliseconds).

Said that, we could do something like this in our `init.lua`.

```lua
-- The code inside that function will be called
-- after everything else in your `init.lua`
vim.defer_fn(function()
  -- Require our plugins declaration module
  require("plugins")

  -- Re-enable syntax highlighting and filetype plugin
  -- once Neovim is fully loaded.
  vim.cmd([[
    syntax on
    filetype on
    filetype plugin indent on
  ]])

  -- This conditional ensures packer and treesitter plugin are
  -- installed before trying to load treesitter plugin
  if packer_plugins and packer_plugins["nvim-treesitter"] then
    vim.cmd("PackerLoad nvim-treesitter")
  end
end, 0)
```

> **Note**: This also brings us a "security" improvement because `vim.defer_fn` function also waits for
> the Neovim API to be safe to call as the Neovim docs says.

#### Is it a good idea to defer everything?

As a short answer, no. It's not a good idea to defer everything.

As I told before it just a timer that delays code execution and can cause issues
if we load some stuff in the wrong order.
