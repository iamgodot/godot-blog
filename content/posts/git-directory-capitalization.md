---
title: "Change the Capitalization of a Direcotry with Git"
date: 2024-05-19T22:51:26-07:00
draft: false
categories:
  - Code
tags:
  - git
---

Today I ran into a small issue with Git when I was trying to capitalize a directory.

Normally this should change the status of the repo so that I can make a commit, while Git just doesn't seem to pick up on it. After a little searching, I found the best way to resolve this was a simple config. According to Git's documentation:

> `core.ignoreCase`

> If true, this option enables various workarounds to enable Git to work better on filesystems that are not case sensitive, like FAT. For example, if a directory listing finds "makefile" when Git expects "Makefile", Git will assume it is really the same file, and continue to remember it as "Makefile".

> The default is false, except git-clone[1] or git-init[1] will probe and set core.ignoreCase true if appropriate when the repository is created.

I didn't set this config globally so it was supposed to be `false`, yet with the above info it indicates that Git will probe and set the config `true` for better working on case-insensitive file systems, such as Apple File System(APFS) on my Mac.

I checked a few repos that I created or cloned, which are all following the same rule with `core.ignoreCase` as `true`. This is a reasonable logic of setting up a Git repo in my mind, next time you can just manually switch the config if running into the same case, after all it should happen very rarely.

---

### References

- [Git config - core.ignoreCase](https://www.git-scm.com/docs/git-config/2.14.6#Documentation/git-config.txt-coreignoreCase)
- [git core.ignoreCase = false in Mac OS X](https://stackoverflow.com/questions/52369109/git-core-ignorecase-false-in-mac-os-x)
