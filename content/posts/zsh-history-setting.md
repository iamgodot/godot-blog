---
title: "History Setting in Zsh"
date: 2024-06-29T22:05:55-07:00
draft: true
categories:
  - Code
---

If you're using zsh, you've probably bundled it with oh-my-zsh, which does a lot of zsh configs underneath. In my case, it has worked well for a pretty long time until I switched to starship, yet it took me a while to figure out how to properly set up history in zsh without it.

First of all, if you're using MacOS, you'll notice things still work fine, this is because there is a system-wide history setting in `/etc/zshrc`. But if you're on Linux, you'll need to set it up yourself.

Without any configs, command history will only be available in the current session, and will be lost once you close it. Besides, different sessions' history won't be shared. To get a better experience, there're 3 options:

1. `setopt appendhistory`: session history will be saved by appending to the end of the history file. But this only happens when you close the session, which is not very handy.
2. `setopt inc_append_history`: what's better about this is an executed command gets immediately written to the history file, thus becomes available in other sessions. However, the current session won't be able to access history from other sessions.
3. `setopt share_history`: besides history writting, this one also reads history for updates, so it's probably the best option in my mind.

One thing to remind is that for history reading you need to fire `history` command, so actions like `CTRL-R` from Fzf just won't work.

In addition to the above options, you'll also want to add a few more configs:

- `HISTFILE`: path of the history file, which is usually `~/.zsh_history`.
- `SAVEHIST`: maximum number of history lines to save in the file, this could be a thousand or a billion, depending on your preference.
- `HISTSIZE`: maximum number of history lines to keep in memory, which is usually the same as `SAVEHIST`. This could also be larger than `SAVEHIST` but not the other way around, because you simply can't save more than you have.

There's also `setopt extendedhistory` and `HISTTIMEFORMAT` to add timestamps for those records, but I don't find them very useful so I won't cover them here. You may try `history -E` to take a look.

I've always been aware of this, but it reminded me again how little I know about the shell I'm using. On the other hand, it actually makes sense that we don't need to bother with learning something until we do.

---

### References

- [Better zsh history](https://www.soberkoder.com/better-zsh-history/)
