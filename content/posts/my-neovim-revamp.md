---
title: "My Neovim Revamp"
date: 2023-04-05T20:31:52+08:00
draft: false
categories:
  - Code
tags:
  - neovim
  - neovim plugins
  - neovim from scratch
---

Yeah, I did it again, but for a good reason.

I have been using VSCode for while, which is great with plugins like `GitLens` and `Docker`. Besides, it's really handy to debug code, view all kinds of files and draw various diagrams such as UML. I thought I'd wave my hands to Neovim, however, I found I still need it, a lot.

Not trying to nitpick, but I have to switch between keyboard and mouse from time to time with VSCode, which sometimes can really interrupt my train of thoughts. In addition, it doesn't feel effortless to use terminal as when I'm in a Tmux session.

For whatever it's worth, I spent some hours on picking Neovim back and upgrading everything. Now it feels unprecedently smooth and efficient, and I'd like to share how I revamped my configs.

# Try it out

First and most important of all, a showcase:

![](https://static.iamgodot.com/content/images/202304071349531.png)

Of course, you will need `Neovim` + `Git` + `Lua` installed. It's better to have `black` + `isort` + `ruff` executables since I use them for Python linting and formatting.

Then replace your `~/.config/nvim` with my [`nvim` folder](https://github.com/iamgodot/dotfiles/tree/master/nvim) and open with `nvim` to see how everything goes.

If it works as expected, `lazy.nvim` is going to take over and install everything shortly, then you're all set.

![](https://static.iamgodot.com/content/images/202304061355014.png)

In case things go south, please leave a comment and let me know. Now if you're still interested, read along and I'll explain as much I can.

# Why Neovim

So I ask myself, why don't I just use Vim? Let's see how Neovim states for itself:

![](https://static.iamgodot.com/content/images/202304061324529.png)

I notice documented API&compatible with Vim, and I'm interested in that builtin terminal emulator, but what really got me is the Lua support for config and plugins. To me, it feels lighter and simpler to use and learn, after all, it's a real programming language. From the [Lua guide](https://neovim.io/doc/user/lua-guide.html) it provides, we can easily pick up:

- Neovim supports a `init.lua` config file with which you can replace `init.vim`.
- Lua uses `require()` to load other modules, so you can do `require(foo.bar)`.
- `foo.bar` or `foo/bar` means a sub-module `bar` under module `foo`.

# My configs

Here is my structure of my config files:

![](https://static.iamgodot.com/content/images/202304061307275.png)

To simply put it, the `init.lua` loads `lazy.lua`, which utilizes `lazy.nvim`, a plugin manager to install and setup every other plugin inside `plugins`. Besides, it will also import from `config` to configure Vim options and keymaps.

# My plugins

It takes a proper combination of plugins to have excellent experiences with Neovim, which is also the best part to enjoy while setting up.

### Plugin manager

I've used `vim-plug` for a very long time while now the popular one is `lazy.nvim`. So I jumped right into it.

It provides a nice UI where you can manage all the packages, and I like how everything gets explained thoroughly in its documents, which is very friendly for starters, even those who are not.

In fact there's even a full-fledged setup repo based on this plugin manager, called `LazyVim` , so that you can build your own on top of it.

### Fuzzy finder

Besides coding, we're always searching stuff, therefore a powerful fuzzy finder is vital. There were `ctrlp` and `fzf`, and now it is `telescope`.

It has pickers, sorters and previewers. So you can find files, do live grep, and search through nearly everything such as help documents, git commits and command history. Just imagine away.

BTW, this could become a test for your keymap design skill. Personally I tried my best to make use of all the leader key combinations.

### LSP support

This outght to be the most complex part, so I'll try to make the best sense of it and explain logically.

First thing to know is LSP requires both client and server, in order to provide code functions like auto-completion. Neovim already has builtin LSP client so we just need to install servers(for our languages) and configure properly.

Therefore we introduce `mason`, which plays a role of language server installer. Just type `:Mason` and install accordingly. This is convenient since we no longer have to prepare executables ourselves, e.g. `pip install --user black`. Pure installation won't suffice, sometimes we need a bit more integration.

For LSP configuration, `mason-lspconfig` allows us to work easier with `nvim-lspconfig`. Let's look at some setup code:

```lua
...

local servers = {
    pyright = {},
    lua_ls = {},
}

local capabilities = vim.lsp.protocol.make_client_capabilities()
capabilities = require("cmp_nvim_lsp").default_capabilities(capabilities)

require("mason").setup()

local mason_lspconfig = require("mason-lspconfig")

mason_lspconfig.setup({
    ensure_installed = vim.tbl_keys(servers),
})

mason_lspconfig.setup_handlers({
    function(server_name)
        require("lspconfig")[server_name].setup({
            capabilities = capabilities,
            on_attach = on_attach,
            settings = servers[server_name],
        })
    end,
})
```

Basically there're two things. One is keep servers installed by `ensure_installed`, the other is setup handlers for them. I only add a default handler yet dedicated ones can be set for certain languages. Moreover, `on_attach` is usually for keymapping and `capabilities` comes in handy for broadcasting to server with additional functionality such as snippets, you can check from `lspconfig` document.

For linters and formatters, `null-ls` is a plugin which makes it every easy to setup these utilities. I setup classic `black`, `isort` as well as the new hot `ruff`, and for Lua I picked `stylua`, which seems just as opinionated as `black`.

Now we have linting and formatting, and we can go to difinition etc. There's only one missing piece left called completion. To achieve that we just need `nvim-cmp` along with some helpers. Personaly I use `LuaSnip` with `friendly-snippets` as its source, and `nvim-path` for file path completion.

So far we've got everything covered for code functions, in which each part can be further extended to support more languages and richer functionalities.

### Miscellaneous

There's no reason to stop here, because so many more await. I'll list some of them here:

- `nvim-tree`: file explorer.
- `vim-fugitive`: git helper.
- `nvim-autopairs`: simple yet necessary.
- `trouble`: manage diagnostics in a better way.
- `lightspeed`: my favorite for faster movements.
- `vim-airline` + `tmuxline`: decorate status lines for both Neovim and Tmux.

Last but not least, you deserve nice color schemes, here's my recommendation:

- `tokyonight`: good and dark.
- `catppuccin`: pretty and elegant.
- `neosolarized`: classic forever.
- `space-vim-dark`: I used it all the time.
- `rose-pine`: Heard they're for minimalist.

# At last

In my mind, using Neovim is all about fast(well, maybe not themes), and once you make it your own, no one can ever steal that from you. I feel so safe with it.

---

*References*

- [My dotfiles](https://github.com/iamgodot/dotfiles)
