---
title: "A Rational Upgrade for My Dotfiles"
date: 2024-07-24T11:06:41-07:00
draft: false
categories:
  - Code
tags:
  - git
---

Recently I've refined my dotfiles which worked out pretty well, while what I'd like to share is not the upgrade, but rather something behind it.

# What I need

This is so cruicial yet so hard that you might realize you're diverting from it when working on the "how" part and have to start all over again. So seriouly, figure it out as much as you can in the first place.

For my case, it's rather simple:

1. Maintain all the dotfiles.
2. Bootstrap on new machines so they'll feel like an old one.

To be more specific, the bootstrap is about the developing environment such as Shell and Neovim other than some niche GUI settings especially on Mac. Moreover, the OSes I care about are pretty much just Arch and MacOS, which reduces a lot of the effort for me to adapt to other Linux distros or even WSL.

Now I can easily imagine my use case in reality:

- On a new machine, with one(or a few) command, every coding tool that makes me feel secure works.
- Every now and then I can cd into the dotfile folder to commit and push the changes.

And this leads to the final satisfying refinement for me so far. However, I've got confused and deflected as well along the path.

# Bump into Nix

If you don't know what Nix is, no worries, I'm not sure if I do either. What I can say is it's a powerful package manager, and there's even an OS called NixOS. But the problem with a powerful tool is you'll likely end up with more chaos than efficiency.

It's not like that I didn't try. Just with the confusing documentation and all the functional programming concepts, the more time I spent, the more overwhelmed and lost I got. In the end I nearly forgot what I need initially and decided to give up. I also read a comment on Reddit that this guy has used NixOS for years yet still got no idea how it works, which is funny yet a little sad too.

I'm not against powerful or complex tools but the lesson I learned is we all should assess their documentations before adopting. User manuals really are getting underestimated by not only users, but also developers.

A more important factor for me to give up than its difficulty is the overkill. It took me a while to realize Nix is not a good match for my simple needs, which is exactly a case of [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it):

> You aren't gonna need it.

# Keep it simple

After Nix I found I can just use shell scripts and a Makefile to achieve the bootstrapping, and turns out it's not as stupid as I thought. Essentially it's like this:

- On a new machine I clone the repo locally.
- There's a simple command from Makefile for bootstrapping.
  - Install a list of packages.
  - Create symlinks of dotfiles.
  - Configure everything else.
- There're also atomic commands for installing or symlinking.

The whole thing was quick and smooth after I started to follow this approach. And it also helps me sort out every process in a very organized way.

# It never ends

This upgrade won't be the last as my needs are inevitably evolving, thus I'll keep learning and exploring.

Besides symlinking, there's also a classic approach taking advantage of a bare Git repo, which is a repo containing only the Git stuff in the `.git` folder from a normal Git repo. The reason to use it is we don't need additonal copies of our dotfiles since they should reside in different folders of home directory. Since a bare Git repo doesn't have a work tree, we can load the entire home directory for it. Then by creating an alias of a special Git command, we are able to manage scattered dotfiles in this bare repo. You can probably peek through this smart method via the following commands:

```bash
git init --bare ~/.dotfiles
alias dotfiles='/usr/bin/git --git-dir="$HOME/.dotfiles/" --work-tree="$HOME"'
dotfiles config status.showUntrackedFiles no
```

Nonetheless, neither this approach or my current one should suffice for some normal features nowadays, to illustrate:

1. Secret management
2. Testing and debugging
3. File encryption

Therefore, I'm actively gathering my potential needs and exploring new dotfile tools. For now I think [chezmoi](https://www.chezmoi.io/) looks pretty promising.

In a nutshell, the key to satisfaction is match your true needs with a proper approach. Let the tools serve the purpose rather than the other way around.

---

### References

- [My dotfiles](https://github.com/iamgodot/dotfiles)
- [Dotfiles - ArchWiki](https://wiki.archlinux.org/title/Dotfiles)
- [What is a bare Git repo and why you need them](https://www.youtube.com/watch?v=8aZW9mYOxhc&ab_channel=EngineerMan)
