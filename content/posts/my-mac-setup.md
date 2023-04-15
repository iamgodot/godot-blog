---
title: "My Mac Setup"
date: 2023-04-13T17:45:41+08:00
draft: false
categories:
  - Code
tags:
  - How to set up a macbook
  - Python for mac
---

Thanks to my lovely gf, I have been able to get my hands on an Apple M2 lately. Coming from Arch with i3, I thought I'd get to this more UI-based and mouse-involved operating system, which seems like a degradation. However, after some digging, it becomes really comfortable for me being both elegant and efficient.

![](https://static.iamgodot.com/content/images/202304151805299.png)

# General setup

I would skip the system settings part, where you may want to customize display, input methods, or modifier keys etc.

### Package manager

First of all, we need a proper tool to be in charge of all the installations, and Homebrew has been the de facto package manager for macOS since a very long time ago.

It's worth noting that since 4.0.0, Homebrew finally ditches its intolerable low-speed updating by switching to JSON:

> Today, I’d like to announce Homebrew 4.0.0. The most significant change since 3.6.0 enables significantly faster Homebrew-maintained tap updates by migrating from Git-cloned taps to JSON downloads.

Which reminds me that this used to be a major reason for me not liking macOS. It always tried to update before executing every command, which was extremely inefficient.

In addition, there's a new `adopt` option for `brew install` now, hence if you have already downloaded and installed something, you could let Homebrew take it over by `brew install --adopt $APP`.

Now that we're ready to install applications, I'd like to share my common list:

- Launcher: Raycast. I use this instead of Alfred.

- Note-taking

  - Logseq: I keep everything here.

  - Obsidian: some features are amazing, e.g. the canvas drawing.

  - Notion: great choice when it comes to sharing and co-working.

- Blogging

  - MarkText: good replacement for Typora.
  - PicGo: useful for uploading images to cloud storage.
- Mind mapping: MindNode. You can't install this by Homebrew, but it's definitely worth the trouble. The above mind map was made by this.

- Media player: IINA. Open source, modern and smooth.

- Screenshot: Shottr. This supports scrolling screenshot.

### Launcher

A good launcher is key to be efficient. Raycast comes with:

- Launching

  - Applications.

  - File search.

  - Online search.

- Clipboard history: supports images as well.

- Floating note: very useful for quick noting.

- Window management: easily arrange every window.

- Utils

  - Check other timezones.

  - Check currency rates.

  - Calculations.

It also has many extensions, I mainly use:

- GitHub

- Easy dictionary

- Emoji search

Besides, you can set alias and hotkeys for accessing all these functions to be just one or two keystrokes, such as `f` for file finding or `g` for Google searching.

Finally, you can export and import a config file from Raycast, although for now they have to be done manually.

# Dev setup

Now that we finished everything as normal people, it's time to open a terminal and play.

### Before diving

We need a good toolkit to start with, such as a friendly shell and a powerful editor.

To start with, I use Alacritty as my terminal emulator. It's fast, and another good thing about it is you can customize by a simple config file with instant effect upon saving. See [here](https://github.com/alacritty/alacritty/blob/master/alacritty.yml) for detailed configuration options.

On top of that, I always choose the classic combination:

1. Zsh: I use Ohmyzsh for its configuration and Powerlevel10k as theme. There are also nice alternatives, e.g. [Starship](https://github.com/starship/starship).

2. Tmux: the terminal multiplexing is all good, except when restarting the machine, which makes [Tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) super useful.

3. Neovim: check out my last post [My Neovim Revamp](./my-neovim-revamp.md).

Now we can head out and prepare for coding, and I'm going to present for Python.

### Python

MacOS has builtin Python installed, which we can call System Python. Besides, we need Dev Python, so that we can keep the former out of the hands of our projects.

For System Python, we leave the original one which resides in `/usr/bin/python3` alone by introducing an installation via Homebrew. This System Python by Homebrew locates at `/opt/homebrew/bin/python3`, which in fact is a link and finally points to somewhere in the Cellar, e.g. `/opt/homebrew/Cellar/python@3.11/3.11.3/Frameworks/Python.framework/Versions/3.11/bin/python3.11`.

For Dev Python, we simply leverage on Pyenv. It allows us to keep multiple versions of Python and easily switch among them by commands, environment variables or special `.python-version` files.

You can read [this blog](https://laike9m.com/blog/best-python-development-setup-for-2022-and-beyond,144/) for more on this topic.

### CLI tool

Sometimes we want command line tools and we want them to be globally available, such as `http` by Httpie. In this case we should use Pipx, of which the slogan states:

> Install and Run Python Applications in Isolated Environments.

It spares virtual environments for each application installed thus we can manage them effortlessly without worrying about dependency conflict.

### Virtual environment

Finally we come to our projects. We choose a proper Python version via Pyenv, and the next step is to create a virtual environment.

There're a lot of choices, and they're still growing. I've tried the following:

- Virtualenv

- Pipenv

- Pew

- Poetry

- PDM

Essentially a good one should satisfy 2 needs:

1. Package CRUD: installation, uninstallation, upgrade etc.

2. Version tracking: via whether `requirements.txt` or some other kind of lock files.

If you're contributing to other projects, you may have to adapt to their selection. For myself, I use PDM currently for a few good reasons:

- It's based on a mature project and well-maintained.

- It's designed as smart and handy with elegant APIs.

- It's documented thoroughly.

- It supports [PEP 582](https://peps.python.org/pep-0582/).

### REPL

Python developers are lucky to have IPython, and Jupyter levels things up further with web-based notebooks. Since JupyterLab depends on IPython, we can put them together:

```bash
# This will expose ipython executable as well
pipx install jupyterlab --include-deps ipython

# Install a better dark theme in the same virtual environment
pipx inject jupyterlab jupyterlab_materialdarker
```

# At last

The above may seem a little overwhelming, and we certainly don't want to repeat it every time on a new machine. I wrap things up in my [dotfiles](https://github.com/iamgodot/dotfiles) so that I can setup with just one command. Take a look for all the softwares and tips I mentioned.

---

*References*

- [macOS Setup Guide](https://sourabhbajaj.com/mac-setup/)

- [Best Python Development Setup for 2022 and Beyond - laike9m's blog](https://laike9m.com/blog/best-python-development-setup-for-2022-and-beyond,144/)
