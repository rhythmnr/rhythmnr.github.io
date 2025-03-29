---
title: git branch m git checkout u
date: 2022-2-3 00:00:00
tags:
- 原创
categories:
- git
---

git branch -m

git branch -m [old] new 重命名分支，将old重命名为new，old可以不填，不填默认重命名当前分支，例子如下：

```shell
   master  git branch -m main                                 ✔  6288  16:55:23
   main  git branch    
```

原来分支名是master，现在叫main了

git push -u

今天用到了一个`git push -u`的命令，完整的是这样写的：

```shell
 git push -u origin main1    
```

然后想看看这-u啥意思，用 git push --help 看看，发现

```shell
GIT-PUSH(1)                                                        Git Manual                                                       GIT-PUSH(1)



NAME
       git-push - Update remote refs along with associated objects

SYNOPSIS
       git push [--all | --mirror | --tags] [--follow-tags] [--atomic] [-n | --dry-run] [--receive-pack=<git-receive-pack>]
                  [--repo=<repository>] [-f | --force] [-d | --delete] [--prune] [-v | --verbose]
                  [-u | --set-upstream] [-o <string> | --push-option=<string>]
                  [--[no-]signed|--signed=(true|false|if-asked)]
                  [--force-with-lease[=<refname>[:<expect>]]]
                  [--no-verify] [<repository> [<refspec>...]]
```

可以看到

```shell
[-u | --set-upstream] [-o <string> | --push-option=<string>]
```

**它和--set-upstream一个作用**，都是给你的github仓库增加新分支的。不过需要注意的是你指定的远程仓库的分支名得和你所在分支一样，不然会报错，如下：

```shell
main-test   git push --set-upstream origin  main-test1
error: src refspec main-test1 does not match any
error: failed to push some refs to 'https://github.com/xx/blog.git'
main-test   git push --set-upstream origin  main-test    1 ↵  6297  17:02:28
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
remote: 
remote: Create a pull request for 'main-test' on GitHub by visiting:
remote:      https://github.com/xx/blog/pull/new/main-test
remote: 
To https://github.com/xx/blog.git
 * [new branch]      main-test -> main-test
Branch 'main-test' set up to track remote branch 'main-test' from 'origin'.
```

把--set-upstream换成-u效果一样。

总结下来就是多用命令自带的查看命令看看，其实很多上面就有，不用去往上搜索。
