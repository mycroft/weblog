---
title: "Jujutsu: a practical intro"
date: 2024-10-27T08:36:46+01:00
summary: My intro to jujutsu
tags:
- jj
- git
---

# Intro

[jujutsu, aka jj](https://github.com/martinvonz/jj), is a git-compatible version configuration system that is aiming to be simple and powerful, reusing features from existing other VCS. I've been testing *jj* for a while and I now need to write down how to use this tool into my workflow.

This post sums up basic usage/acts as a cheatsheet.

# Getting a jj repository

## Creating a new repository

As for now, it is required to use `git` as a backend when creating a new repository. `jj init` will just fail.

```sh
$ jj git init
Initialized repo in "."
```

## Checkout of an existing repository from github

```sh
$ jj git clone git@git.mkz.me:mycroft/til.git
Fetching into new repo in "/home/mycroft/tmp/til"
bookmark: main@origin [new] untracked
Setting the revset alias "trunk()" to "main@origin"
Working copy now at: ywksyxpz 1f45511f (empty) (no description set)
Parent commit      : omvnwxrw 1cf9d9f4 main | docs: adding cryptography/basic-rsa-with-python.md
Added 12 files, modified 0 files, removed 0 files

$ cd til/
$ jj show
Commit ID: 1f45511f22ade210213d984e72eae20286cd3150
Change ID: ywksyxpzzmmuxxlwxlxrtnvxtmtqsxyo
Author: Patrick MARIE <pm@mkz.me> (2024-10-27 09:13:04)
Committer: Patrick MARIE <pm@mkz.me> (2024-10-27 09:13:04)

    (no description set)
```

## Using jj on an existing git repository

```sh
$ git clone git@git.mkz.me:mycroft/k8s-home.git
Cloning into 'k8s-home'...
remote: Enumerating objects: 9001, done.
remote: Counting objects: 100% (9001/9001), done.
remote: Compressing objects: 100% (3310/3310), done.
remote: Total 9001 (delta 6574), reused 7669 (delta 5634), pack-reused 0
Receiving objects: 100% (9001/9001), 1.40 MiB | 20.52 MiB/s, done.
Resolving deltas: 100% (6574/6574), done.

$ cd k8s-home/
$ jj git init --git-repo .
Done importing changes from the underlying Git repo.
Setting the revset alias "trunk()" to "main@origin"
Hint: The following remote bookmarks aren't associated with the existing local bookmarks:
  main@origin
Hint: Run `jj bookmark track main@origin` to keep local bookmarks updated on future pulls.
Initialized repo in "."
```

# Looking at log

Once you've got a `jj` repository, you can check its history and check single revision details:

```sh
$ jj log
@  qsnqopks pm@mkz.me 2024-10-27 09:09:12 64e66208
│  (empty) (no description set)
◆  zzzzzzzz root() 00000000

$ jj show -r qsnqopks
Commit ID: 64e662080019bd8178b992ffc8ad3923ff44059b
Change ID: qsnqopksxwrostxuuxomkmpousqrrrly
Author: Patrick MARIE <pm@mkz.me> (2024-10-27 09:09:12)
Committer: Patrick MARIE <pm@mkz.me> (2024-10-27 09:09:12)

    (no description set)
```

Note: On pushed repositories/external retrieved repositories, the history can show immutable revisions. `jj log` will only show history to latest immutable revision. If you need to see older revision, you'll need to tweak the `jj log` query.

```sh
$ jj log -r ..@
@  ywksyxpz pm@mkz.me 2024-10-27 09:13:04 1f45511f
│  (empty) (no description set)
◆  omvnwxrw pm@mkz.me 2024-10-02 10:53:06 main 1cf9d9f4
│  docs: adding cryptography/basic-rsa-with-python.md
◆  zmtsywzy pm@mkz.me 2024-09-30 11:51:33 c3b29bf7
│  docs: updating security/decrypt-ntlm-v2-hmac-md5-hashes.md
◆  osrlutwm pm@mkz.me 2024-09-30 11:49:43 3f8def91
│  docs: adding security/decrypt-ntlm-v2-hmac-md5-hashes.md
...
```

`..@` means "all changes until now - @". Another example:

```sh
$ jj log -r osrlutwm-..
@  ywksyxpz pm@mkz.me 2024-10-27 09:13:04 1f45511f
│  (empty) (no description set)
◆  omvnwxrw pm@mkz.me 2024-10-02 10:53:06 main 1cf9d9f4
│  docs: adding cryptography/basic-rsa-with-python.md
◆  zmtsywzy pm@mkz.me 2024-09-30 11:51:33 c3b29bf7
│  docs: updating security/decrypt-ntlm-v2-hmac-md5-hashes.md
◆  osrlutwm pm@mkz.me 2024-09-30 11:49:43 3f8def91
│  docs: adding security/decrypt-ntlm-v2-hmac-md5-hashes.md
~
```

`osrlutwm-..` means all commits from `osrlutwm`'s parent commit (-).


## Creating revisions

Now that we can create a repository and view its history, it is time to create some diff! Contrary to git, you don't "commit" changes. The way it is working can be a bit different: You're working on a current "working copy" and once you're done, you create a new one! The basic workflow is:

- You set a description to the current working copy (remember when we `jj show` ? it had no description)
- You work on changes
- Once happy, you create a new working copy based on your changes!

```sh
$ jj show
Commit ID: ca7f82bb1c3cac24f0615ce49f371f41574944d3
Change ID: qsnqopksxwrostxuuxomkmpousqrrrly
Author: Patrick MARIE <pm@mkz.me> (2024-10-27 13:06:40)
Committer: Patrick MARIE <pm@mkz.me> (2024-10-27 13:10:29)

    (no description set)

$ jj describe -m "feat: create a new project based on zig"
Working copy now at: qsnqopks bbe7c14f feat: create a new project based on zig
Parent commit      : zzzzzzzz 00000000 (empty) (no description set)

$ zig init
info: created build.zig
info: created build.zig.zon
info: created src/main.zig
info: created src/root.zig
info: see `zig build --help` for a menu of options
$ echo zig-out >> .gitignore
$ echo .zig-cache >> .gitignore

$ jj status
Working copy changes:
A .gitignore
A build.zig
A build.zig.zon
A src/main.zig
A src/root.zig
Working copy : qsnqopks bbe7c14f feat: create a new project based on zig
Parent commit: zzzzzzzz 00000000 (empty) (no description set)

$ jj new
Working copy now at: sqvpnnot fe034d70 (empty) (no description set)
Parent commit      : qsnqopks bbe7c14f feat: create a new project based on zig

$ jj log
@  sqvpnnot pm@mkz.me 2024-10-27 13:13:08 fe034d70
│  (empty) (no description set)
○  qsnqopks pm@mkz.me 2024-10-27 13:12:01 bbe7c14f
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

To wrap up:

- We set a description with `jj describe -m "..."`
- Once done & happy with out changes, we create a new working copy with `jj new`

and that's it! We now have a new working copy we can work on.


## Branches?

Now, let's create some diff and new change set:

```sh
$ jj describe -m "feat: changing print statement"
Working copy now at: sqvpnnot 8038ae5e feat: changing print statement
Parent commit      : qsnqopks bbe7c14f feat: create a new project based on zig

$ sed -ie 's/belong to us./belong to me./' src/main.zig

$ jj new
Working copy now at: pynrzmnr e5bec6be (empty) (no description set)
Parent commit      : sqvpnnot 8038ae5e feat: changing print statement

$ jj log
@  pynrzmnr pm@mkz.me 2024-10-27 13:17:16 e5bec6be
│  (empty) (no description set)
○  sqvpnnot pm@mkz.me 2024-10-27 13:17:07 8038ae5e
│  feat: changing print statement
○  qsnqopks pm@mkz.me 2024-10-27 13:12:01 bbe7c14f
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

What if we now consider this change should be part of a branch and want to go back in time? Well, there is `jj bookmark` for this:

```sh
$ jj bookmark create statement-change -r @-
@  kqsutqqu pm@mkz.me 2024-10-27 13:49:56 9221126d
│  (empty) (no description set)
○  sqvpnnot pm@mkz.me 2024-10-27 13:46:51 statement-change 4b93825b
│  feat: changing print statement
○  qsnqopks pm@mkz.me 2024-10-27 13:46:51 1a18916b
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

Note: I kinda modified modified `qsnqopks` so it changed commit ids. It does not really matter.

As I now have a bookmark on that change, let's create a new branch based on `qsnqopks`:

```sh
$ jj new -r qsnqopks
Working copy now at: mnorzwky 17472305 (empty) (no description set)
Parent commit      : qsnqopks 1a18916b feat: create a new project based on zig
Added 0 files, modified 1 files, removed 1 files

$ jj log
@  mnorzwky pm@mkz.me 2024-10-27 13:52:48 851df1bd
│  (no description set)
│ ○  sqvpnnot pm@mkz.me 2024-10-27 13:46:51 statement-change 4b93825b
├─╯  feat: changing print statement
○  qsnqopks pm@mkz.me 2024-10-27 13:46:51 1a18916b
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

And I'll modify the same `src/main.zig` file so I'll create a conflict for the future:

```sh
$ jj diff
Modified regular file src/main.zig:
   5    5:     std.debug.print("All your {s} are belong to usthem.\n", .{"codebase"});
    ...

$ jj st
Working copy changes:
M src/main.zig
Working copy : mnorzwky 851df1bd (no description set)
Parent commit: qsnqopks 1a18916b feat: create a new project based on zig

$ jj describe -m "feat: another  print statement"
Working copy now at: mnorzwky 623fe0ea feat: another  print statement
Parent commit      : qsnqopks 1a18916b feat: create a new project based on zig

$ jj new
Working copy now at: kpwklolq bd2ff65b (empty) (no description set)
Parent commit      : mnorzwky 623fe0ea feat: another  print statement
```

So now, what if I want to merge `statement-change` in current branch/working copy?

We have our 2 heads:

```sh
$ jj log -r 'heads(all())'

@  kpwklolq pm@mkz.me 2024-10-27 14:04:30 bd2ff65b
│  (empty) (no description set)
~

○  sqvpnnot pm@mkz.me 2024-10-27 13:46:51 statement-change 4b93825b
│  feat: changing print statement
~
```

And we merge them into a new changeset:

```sh
$ jj new kpwklolq sqvpnnot -m "merge"
Working copy now at: trxrwzzr 2479582f (conflict) (empty) merge
Parent commit      : kpwklolq bd2ff65b (empty) (no description set)
Parent commit      : sqvpnnot 4b93825b statement-change | feat: changing print statement
Added 1 files, modified 1 files, removed 0 files
There are unresolved conflicts at these paths:
src/main.zig    2-sided conflict
```

Ah, there is a conflict! It is just required to fix it in the working copy and once done, everything will be fine:

```sh
$ jj st
Working copy changes:
M src/main.zig
Working copy : trxrwzzr 3d1845dd merge
Parent commit: kpwklolq bd2ff65b (empty) (no description set)
Parent commit: sqvpnnot 4b93825b statement-change | feat: changing print statement

$ jj log
@    trxrwzzr pm@mkz.me 2024-10-27 14:10:02 3d1845dd
├─╮  merge
│ ○  sqvpnnot pm@mkz.me 2024-10-27 13:46:51 statement-change 4b93825b
│ │  feat: changing print statement
○ │  kpwklolq pm@mkz.me 2024-10-27 14:04:30 bd2ff65b
│ │  (empty) (no description set)
○ │  mnorzwky pm@mkz.me 2024-10-27 14:04:28 623fe0ea
├─╯  feat: another  print statement
○  qsnqopks pm@mkz.me 2024-10-27 13:46:51 1a18916b
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

Let's do some cleaning. First, there is this empty `kpwklolq` revision that we can just kill it with `jj squash`:

```sh
$ jj squash  -r kpwklolq
Rebased 1 descendant commits
Working copy now at: trxrwzzr 1a9eec8d merge
Parent commit      : mnorzwky 46a6e500 feat: another  print statement
Parent commit      : sqvpnnot 4b93825b statement-change | feat: changing print statement

$ jj log
@    trxrwzzr pm@mkz.me 2024-10-27 14:13:37 1a9eec8d
├─╮  merge
│ ○  sqvpnnot pm@mkz.me 2024-10-27 13:46:51 statement-change 4b93825b
│ │  feat: changing print statement
○ │  mnorzwky pm@mkz.me 2024-10-27 14:13:37 46a6e500
├─╯  feat: another  print statement
○  qsnqopks pm@mkz.me 2024-10-27 13:46:51 1a18916b
│  feat: create a new project based on zig
◆  zzzzzzzz root() 00000000
```

## Pushing changes

We're going to use git remotes to share work with github. Let's define some origin:

```sh
$ jj git remote add origin git@github.com:mycroft/jj-test.git

$ jj new
Working copy now at: knrwxmuv 56044f52 (empty) (no description set)
Parent commit      : trxrwzzr 1a9eec8d merge

$ jj git push -c @-
Creating bookmark push-trxrwzzrkpnp for revision trxrwzzrkpnp
Changes to push to origin:
  Add bookmark push-trxrwzzrkpnp to 1a9eec8d6d0a
remote: Resolving deltas: 100% (6/6), done.

$ jj bookmark list
push-trxrwzzrkpnp: trxrwzzr 1a9eec8d merge
statement-change: sqvpnnot 4b93825b feat: changing print statement
```

This will push an anonymous branch to github.

See [Working with Github](https://martinvonz.github.io/jj/latest/github/)


# External resources

- [Official docs](https://martinvonz.github.io/jj/latest/)
- [Official tutorial](https://martinvonz.github.io/jj/latest/tutorial/)
- [Steve Klabnik's jujutsu tutorial](https://steveklabnik.github.io/jujutsu-tutorial/)
- [jj init, by Chris Krycho](https://v5.chriskrycho.com/essays/jj-init/)
- [jj in practice, by Arne Bahlo](https://arne.me/blog/jj-in-practice)
- [https://tonyfinn.com/blog/jj/](https://tonyfinn.com/blog/jj/)