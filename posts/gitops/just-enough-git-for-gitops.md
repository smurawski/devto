---
title: Just Enough Git for GitOps
description: 'GitOps Series   This week and next, my team and I are going to be exploring different facets...'
tags: 'git, gitops, cloudnative, beginners'
published: true
id: 1604018
date: '2023-09-18T16:47:35Z'
---

## GitOps Series

This week and next, my team and I are going to be exploring different facets of GitOps and some of the tools in the space. Watch our social media for new posts:

- Josh Duffney: [twitter](https://twitter.com/joshduffney)|[linkedin](https://www.linkedin.com/in/joshduffney/)|[github](https://github.com/duffney)
- Paul Yu: [twitter](https://twitter.com/pauldotyu)|[linkedin](https://www.linkedin.com/in/yupaul/)|[github](https://github.com/pauldotyu)
- Jorge Arteiro: [twitter](https://twitter.com/JorgeArteiro)|[linkedin](https://www.linkedin.com/in/jorgearteiro/)|[github](https://github.com/jorgearteiro)
- Yosh Wuyts: [twitter](https://twitter.com/yoshuawuyts)|[linkedin](https://www.linkedin.com/in/yoshuawuyts/)|[github](https://github.com/yoshuawuyts)
- Steven Murawski (me): [twitter](https://twitter.com/stevenmurawski)|[linkedin](https://www.linkedin.com/in/usepowershell/)|[github](https://github.com/smurawski)

Some of the topics we'll cover include:

- What is GitOps?
- How do Secure Supply Chain tools fit in a GitOps workflow?
- FluxCD
- ArgoCD
- Dealing with secrets in GitOps

What else would you want to see? Leave a comment below with topics you'd like to see covered and we'll see what we can do!

Let's get started with some concepts about Git and then cover a basic set of commands that will get you ready to work with GitOps tools.

## Git Concepts

Git is a distributed version control system that allows multiple developers to collaboratively manage and track changes to source code, enabling efficient code development, and review.

{% embed https://www.youtube.com/watch?v=JlpRfMRWNY8 %}

### Distributed

The distributed nature of Git means that you can have multiple independent copies of a repository with some shared history. That shared history makes it possible to push or pull changes from other copies effectively.

What this effectively means is that Git does not, by design, have the concept of a central repository. All copies of a repository are just as valid as any other. We, as developers and operations personnel, enforce authority to one particular copy of a repository by pattern and practice. Tools and services like GitHub, GitLab, Azure Repos, and others help facilitate that practice.

For the purposes of GitOps, this means that we enforce the source of truth for our environment by convention and process.

### Branches

In Git, branches are separate workspaces that allow developers to work on features, bug fixes, or experiments independently from other changes.

This makes it easier to manage and merge changes into the project when they are ready.

By convention, the primary branch in a repository is `main`. Older projects may use the `master`, though that terminology is perceived as culturally insensitive or offensive and the industry has shifted towards other, equally accurate terminology. Another term you may see for the `main` branch is `trunk`.

{% embed https://www.youtube.com/watch?v=rv8HFYNmHG8 %}

### Repository

Repositories in Git are represented as the directory structure of the project under control. There is a hidden `.git` directory that handles all the change tracking and content across branches and tags represented in the repository.

**Repo** is a common shorthand version of repository.

### Special files

As I previously mentioned, a Git repository has a special directory for tracking Git related operations - the `.git` directory.

One other key file to be aware of at the start is the `.gitignore` file. This file allows you to define specific files or groups of files (wildcards supported) that Git should ignore. This is commonly used to keep Git from tracking specific binary files, intermediate build artifacts, local settings files, and editor specific files - though it can be used for any file or set of files you do not want tracked.

### Remotes

Remotes are repositories that are in a physically different location (either on the local filesystem or available over a network). Git can track named references to these other locations.

By convention, if you copy (clone) a repository from a remote source, that original source is tracked as a remote named `origin`.

{% embed https://www.youtube.com/watch?v=3Brg_l0vHBo %}

## Commands

These are the commands, in order of use, that I think will be helpful as you get started with Git and GitOps. There are more complicated operations - let me know in the comments below if you'd like to see more specific scenarios with Git.

### Clone

We'll often be starting with a source repository from a central server. One common location is from GitHub. To create a local copy of a repository from a remote source, we use the `git clone` command.

Git can operate over local file system paths or over a network using HTTPS or SSH. We provide a remote source to `git clone` and Git will track that method and use it for future operations with that remote repository.

The cloned remote repository will be tracked as a remote named `origin`.

Example:

HTTPS

```
git clone https://github.com/Azure-Samples/aks-store-demo
```

SSH

```
git clone git@github.com:Azure-Samples/aks-store-demo.git
```

### Init

If you aren't starting from a remote Git repository, you can create a new git repository from just about any directory structure. `git init` will create the tracking data required for the repository (the `.git` folder). There will be no `origin` or other remotes defined.

Example:

```
> mkdir init_sample

    Directory: C:\Users\stmuraws\source

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----           9/18/2023 10:10 AM                init_sample
```

```
> cd init_sample
> git init .
Initialized empty Git repository in C:/Users/stmuraws/source/init_sample/.git/
```

### Status

The most common command I run inside a Git repo is `git status`. This will tell you the current state of changes. You'll see your current branch, status compared to the same branch against any origin you are tracking against (usually `origin`), as well as any pending file changes **staged** by `git add` (described next).

Example (from the `init_sample` repository):

```
> git status
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

### Add

Regardless of how you got your Git repository locally (whether you cloned an existing repo or used `git init` to create a new one), when you add new, remove existing, or change existing files, you need a way to tell Git what you want to track.

Git is not like _autosave_. It does not continuously track changes. We'll tell Git specifically which changes to track.

`git add` is what we'll use to tell Git which changes to **stage** as part of a **commit**.

Example (from the `init_sample` repository):

```
> 'some content' > sample_file.txt
> git add sample_file.txt
> git status

On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   sample_file.txt
```

`git add` does not create a new **commit** by itself. We'll use it in conjunction with the next command, `git commit`.

### Commit

Once we have some changes **staged**, we can commit them to our repository. Each **commit** is identified by a hash.

**Commits** require a message by default. If you do not pass one at the command line or use a flag to allow an empty message, then an editor will open to allow you to type a new commit message. I'll typically use the `-m` parameter to supply a message at the command line.

Example (from the `init_sample` repository):

```
> git commit -m "First commit to the sample repository."

[main (root-commit) 01e9e34] First commit to the sample repository
 1 file changed, 1 insertion(+)
 create mode 100644 sample_file.txt
```

Ideally, each **commit** would be a set of related changes.

### Push

Now that we've made a change, if we are working with a shared remote repository, we can push our changes to that repository.

If we started by cloning a remote repository, then we'll already have a remote configured.

For this example, let's use the repository we used for `git init` (so we don't make random changes to something we might really be using). To use this repository, we'll need to do a bit of setup.

Setup (from the `init_sample` repository):

```
> cd ..
> mkdir push_sample
> cd push_sample
> # Initialize a new git repo as bare (no working directory)
> git init --bare .
> cd ../init_sample
```

Once we are back in the `init_sample` directory, we'll configure the new repository as a remote.

```
> git remote add origin ../push_sample
```

Now, we are ready to push our changes. The remote setup usually only happens once for the repository (and if we cloned a repository, it'll be setup for us already). We'll specify the remote we are pushing to (`origin`) and the branch we are pushing (`main`). The `-u` is only needed once. It tells Git that the `main` branch of my local repository and my remote repository should be associated (**tracked**).

```
> git push origin main -u

Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 254 bytes | 254.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To ../push_sample
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

### Fetch

`git fetch` retrieves the contents and **commits**. It does not change anything in your **working directory**.

I prefer to use `git fetch`, in combination with the next command `git merge`, rather than `git pull` to retrieve changes from a remote repository. This puts me in greater control of the changes that will happen to the files I'm currently working on. `git pull` is convenient until it bites you (which can be a rare occurrence).

Example (from our `init_sample` repository):

```
> cd ..
> mkdir fetch_sample
> cd fetch_sample
> git init .
> git remote add origin ../push_sample
> git fetch origin

remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 234 bytes | 21.00 KiB/s, done.
From ../push_sample
 * [new branch]      main       -> origin/main
```

### Merge

`git merge` allows you to bring changes from one branch into your current branch. The source branches can be local or from a remote (which have been `fetch`ed locally).

Now that we have changes from a remote repository in our local Git repository (from the `git fetch`), we can merge changes from the remote branch to our local branch.

Example (from the `fetch_sample` repository):

```
> # Verify the directory is empty (other than the hidden .git directory)
> ls
> git merge origin/main
> ls

    Directory: C:\Users\stmuraws\source\fetch_sample

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---           9/18/2023 11:09 AM             13 sample_file.txt
```

### Checkout

`git checkout` allows us to switch between branches inside our repository. This allows us to more easily keep our work isolated from other ongoing changes. `git checkout` has a handy switch, `-b`, that lets us create a new branch from the current branch (otherwise, you can use the `git branch` command, but I almost never use that to create new branches since `git checkout` is so convenient).

Ideally, we create a branch for every set of related changes we want to make.

Example (from the `fetch_sample` repository):

```
> git checkout -b 'new_branch'
Switched to a new branch 'new_branch'
> git checkout main
Switched to branch 'main'
```

### Reset

Sometimes, we can get our repositories all messed up. `git reset` can help us. (There are other uses too, but early on this is the most common.)

There are two modes for `git reset`, soft and hard.

#### Soft resets

A soft reset leaves the changes from commits after the specified commit in your working directory, allowing you to retain the changes you made in those commits as uncommitted modifications. Any of the previously **staged** changes (things we added with `git add`) will be no longer be staged, but those changes will still be present as uncommitted modifications.

Use a soft reset when you want to rework and recommit some changes from the commits after the specified commit. Or if you accidentally committed something but want to include those changes in a different commit.

#### Hard resets

A hard reset resets both the **staging** (things we added with `git add`) area and the working directory (any current changes in files) to match the state of the specified **commit**. This means any changes made in **commits** after the specified **commit** are completely removed.

**WARNING**: A hard reset can and will remove/lose changes that are not present in target **commit** you are resetting to.

Use a hard reset when you want to completely discard all changes made in **commits** after a certain point and start fresh.

### Rebase

The last command I'm going to introduce here is `git rebase`. There are two modes of `git rebase` that I commonly use, but we'll focus on the most critical use case.

We can use rebasing to bring our branch history current to the state of whatever branch we want to target. This will make it easier to bring our changes into that target branch and let us solve any conflicts in our working branch locally.

For example, I'm [working on some Helm charts](https://github.com/smurawski/aks-store-demo/tree/helm) for the [AKS Store Demo project](https://github.com/Azure-Samples/aks-store-demo). My working branch is currently behind the state of the project source. I can bring my project current with `git rebase` and make it easier for the project maintainers to accept my contribution.

```
> git clone https://github.com/smurawski/aks-store-demo
> cd aks-store-demo
> git checkout origin/helm
> git checkout -b helm
> git remote add upstream https://github.com/Azure-Samples/aks-store-demo
> git fetch upstream
> git rebase upstream/main
Successfully rebased and updated refs/heads/helm.
```

What about merging the main branch back into my working branch? You can do that, but it can make the history pretty messy and, in some cases, make reviews more difficult (especially when there are a lot of files modified or moved around).

## Summary

This was a very brief introduction to a **lot** of commands, but it should be enough to get started. For a more hands-on exploration of Git, check out [this learning path on Microsoft Learn](https://learn.microsoft.com/training/paths/intro-to-vc-git/).

We didn't cover anything about dealing with merge conflicts, but we get into that a bit in this video or [you can get hands on with Microsoft Learn](https://learn.microsoft.com/training/modules/branch-merge-git)

{% embed https://www.youtube.com/watch?v=rv8HFYNmHG8 %}

### Continue the conversation

Leave your questions in the comments or come over to the Microsoft Open Source Discord and chat with me and my team in the _cloud-native_ channel!

- [Join the Microsoft Open Source Discord](https://aka.ms/cloudnative/JoinOSSDiscord)
- [Meet us in the Cloud Native Channel](https://aka.ms/cloudnative/DiscordChannel)

Watch for a post from our team tomorrow covering what GitOps is!
