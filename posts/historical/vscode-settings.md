---
tags: 'beginner, git, vscode'
title: My Git and VS Code Settings
description: My Git and VS Code Settings
published: true
canonical_url: null
id: 1606437
date: '2023-09-20T19:06:36Z'
---

I work regularly with several different project types and languages across Windows, Mac, and Linux.

I find some basic settings for Git and VS Code smooth out some headaches.

My user settings include:

```
"files.eol": "\n", // Windows usually doesn't care, but Linux/Mac tooling is way more sensitive
"editor.trimAutoWhitespace": true,
"editor.fontSize": 16,
"editor.formatOnPaste": true,
"editor.formatOnSave": true,
"window.zoomLevel": 0,
"[powershell]": {
    "editor.tabSize": 4
},
"[ruby]": {
    "editor.tabSize": 2
}
```

My git config consists of:

```
[credential]
	helper = wincred
[user]
	email = <my email here>
	name = Steven Murawski
	signingkey = <my gpg signing key here>
[diff]
	tool = default-difftool
[difftool "default-difftool"]
	cmd = code --wait --diff $LOCAL $REMOTE
[alias]
	commit = commit -S -s
[core]
	editor = code --wait
	eol = lf
	autocrlf = input
	excludesfile = ~/.gitignore
[difftool "winmerge"]
	cmd = /c/Program\\ Files\\ \\(x86\\)/WinMerge/WinMergeU.exe
[gpg]
	program = c:/Program Files (x86)/GNU/GnuPG/gpg2.exe
```

Since I work on a number of projects that require a [DCO signoff](https://developercertificate.org/), I just default to signing off my commits (`alias.commit`) and I GPG sign them because I like how they look in the GitHub UI. ;)

I've set git to only use Linux line feeds for new clones/fetches (`core.eol`) and change anything with CRLFs on upload (`core.autocrlf`). To aid in this, my editor config has line endings set to `\n` as well.
