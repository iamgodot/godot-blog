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

I checked a few repos that I created or cloned, which are all following the same rule with `core.ignoreCase` as `true`. This is a reasonable logic of setting up a Git repo in my mind, ~~next time you can just manually switch the config if running into the same case~~, after all it should happen very rarely.

_Update:_

After a while I found the above change didn't really solve the problem, as both the lowercased and uppercased directories started to appear in the Git status. I also checked the GitHub repo in which both existed as well.

Turns out the right way to do should be: `git mv --force foo Foo`. And since I've messed up already, I had to clear the cache by `git rm -r --cahed foo` and set `core.ignoreCase` back to be `true`. Then I pushed the commit, and confirmed that remote repo got corrected with only the uppercased directory.

Hope this can clear things up and you don't make the same mistake as I did.

---

### References

- [Git config - core.ignoreCase](https://www.git-scm.com/docs/git-config/2.14.6#Documentation/git-config.txt-coreignoreCase)
- https://stackoverflow.com/questions/6899582/i-change-the-capitalization-of-a-directory-and-git-doesnt-seem-to-pick-up-on-it
