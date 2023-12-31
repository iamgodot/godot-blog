---
title: "Formatting in Neovim"
date: 2023-04-20T16:05:53+08:00
draft: true
categories:
  - Code
tags:
  - Neovim format on save
  - Neovim lsp format
---

> “Mock across architecturally significant boundaries, but not within those boundaries.”

With Neovim and `null-ls` plugin we can make code formatting work like a charm, as I stated in [My Neovim Revamp](./my-neovim-revamp.md) previously. While I was happily enjoying it, I did some tweaking for a little more convenience, such as:

- Format on save.

- Customize formatter.

- Format conditionally in runtime.

Let's get right into it.

# Format on save

This is plain easy if you check out the wiki of `null-ls`. According to [this part](https://github.com/jose-elias-alvarez/null-ls.nvim/wiki/Formatting-on-save#sync-formatting), all you need is copy&paste the code into your own config.

# Customize ruff&stylua

[Formatters](https://github.com/jose-elias-alvarez/null-ls.nvim/blob/main/doc/BUILTINS.md#formatting) supported by `null-ls` all have defaults, like arguments. Sometimes we want to add more options, and that's when `extra_args` comes in handy.

For instance, I'm using `ruff` for Python formatting to replace `isort`, so customization becomes:

```lua
require("null-ls").builtins.formatting.ruff.with({
    extra_args = { "--select", "I" }
})
```

And for Lua, I want `stylua` to use spaces instead of tabs, so I did:

```lua
require("null-ls").builtins.formatting.stylua.with({
    extra_args = { "--indent-type", "Spaces" }
})
```

In addition, you can replace default arguments totally with `args`, by following the config instruction [here](https://github.com/jose-elias-alvarez/null-ls.nvim/blob/main/doc/BUILTIN_CONFIG.md#arguments).

BTW, I found a few tricks regarding tabs&spaces:

- In Neovim:

  - Use `:retab` to replace tabs with spaces(number of spaces defined by `tabstop`).

  - Use `:set list` to show tabs marking as `>` and trailing spaces as `-`.

- In Shell:

  - Use `sed` to convert tabs to spaces in a bunch of files at one time.

  - On Linux: `sed -i "s/\t//g" /path/to/files`.

  - On macOS: `sed -i "" "s/\t//g" /path/to/files`. Here `-i` stands for extension rather than in-place editing.

# Format conditionally

Auto-formatting is sweet, but what if I don't want it? When I'm reading some source code or working on a shared project, I want things kept as they were.

Toggling formatters on&off can be annoying, is there a smarter way?

Yes, there sure is. Turns out `null-ls` has a `runtime_condition` setting to force it to check whether a formatter should run for the current buffer.

Essentially, you just need to add a function to determine if formatting is needed:

```lua
require("null_ls").builtins.formatting.black.with({
    runtime_condition = function(params)
        return string.match(params.bufname, "source") == nil
    end,
})
```

I keep all source code projects in `~/projects/source`. Based on the above config `black` will check if current file belongs under this particular path, and if it is, no auto-formatting will be performed.

Just like this, you can easily split things up with a suitable condition statement in Lua. For reference of how to better write it, see:

- [runtime_condition](https://github.com/jose-elias-alvarez/null-ls.nvim/blob/main/doc/BUILTIN_CONFIG.md#runtime_condition)

- [params](https://github.com/jose-elias-alvarez/null-ls.nvim/blob/main/doc/MAIN.md#params)

Hope these tips can be helpful :)
