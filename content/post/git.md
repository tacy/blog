---
title: "git notes"
date: 2016-01-12
lastmod: 2016-01-12
draft: false
tags: ["tech", "git", "notes"]
categories: ["tech"]
description: "git使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# squash #
在你的work branch完成某个feature之后，你需要merge到master：

``` sh
git checkout master
git merge workbranch
```

这种做法的问题在于，一般你完成某个feature的时候，会做多次commit，而这些commit如果没有实际意义，那么也就没有在master出现的必要，这样的话你就需要用到squash了，squash的意思是，把work branch上的多个commit合并成一个commit，merge到主干上。
``` sh
git checkout master
git merge --squash workbranch
git commit -m "message here"
```
## squash branch
git reset --soft HEAD~3 &&
git commit
git push origin fix33333 -f


# cherrypick
git checkout source-branch
git log  # get commit hash
git checkout target-branch
git cherry-pick <commit hash>

# rebase #
work branch需要和master同步, 可以用到rebase, rebase是git的核心概念[^1]

两个用户同时工作在同一个分支, 如果a用户提交了代码到远端仓库, b用户本地代码库没有和远端仓库同步, 那么b用户需要提交本地代码到远端仓库可以用rebase实现, 当然也能用merge方式:

```
# As before - update your local repo with the latest code:
git fetch --all --prune

# Merge the changes with the rebase flag
git pull --rebase origin branch1

# push to remote repo
git push origin branch1
```

[^1]: [git rebase](https://github.com/jrblevin/markdown-mode)

# branch #

## 常用
`git branch -r / git branch -a / git branch -vv`

## checkout
我希望调试我现在用的某个版本的Kuernetes, 例如1.1.2, 首先我fork kubernetes, 然后checkout到我的仓库, 同时添加远程仓库到我的本地, 保持和远程主仓库同步.

查看主代码库你会发现, 1.1是branch, 1.1.2是tag, 简单说, 1.1.2就是某个revison的名字, 很多人在某个branch上工作, 感觉代码稳定的时候, 在分支上打一个tag, 然后发布这个tag, 后续针对该版本的bug修改, 需要checkout这个tag, 修改, 然后合并到branch上.

首先我们checkout远程主仓库分支到我们本地仓库:
git remote show upstream
git checkout -b 1.1 upstream/release-1.1

然后切换到tag:
git checkout tags/v1.1.2 -b tacy-v.1.1.2

  * 查看当前分支状态: git status
  * 临时切换到其他分支, 不想提交当前一半的工作: git stash / git stash list / git stash apply ( git stash apply stash_name)

分支上建分支:
git checkout master
git checkout -b dev master
git checkout dev
git checkout -b dev-b1 dev

### push local branch to remote
In Git 1.7.0 and later, you can checkout a new branch:

git checkout -b <branch>
Edit files, add and commit. Then push with the -u option:

git push -u origin <branch>
Git will set up the tracking information during the push.

## delete remote branch
`git push origin --delete <branch_name>`  or `git push origin :<branch_name>`
git branch -d <branch_name>


# tag
create tag: `git tag v1.6.2` & `git tag -d v1.6.2`
push tag to remote: `git push --tag`
delete remote tag: `git push origin :v1.6.2`

# github
Github有两种登入方式, 一种是https, 一种是ssh, 推荐ssh方式, 直接配置key就可以了
ssh: git@git@github.com:USERNAME/REPOSITORY.git
https: https://github.com/USERNAME/OTHERREPOSITORY.git

`git remote -V` 可以查看remote response的URL, `git remote set-url origin yoururl`可以修改URL, 在ssh和https之间切换

## sync fork

```
#git remote -v
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
#git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
#git remote -v
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)

#git fetch upstream
#git checkout master
#git merge upstream/master
```

# ignore
编辑`.gitignore`文件, 添加你要忽略的文件, 如果想忽略的文件已经提交进入代码库, 需要收到删除: `git rm --cache FILENAME`, 否则git不会忽略该文件.

## 保留目录, 不提交文件
有时候, 希望保留目录, 但是不希望保留目录里面的内容, 简单做法是在希望保留的目录底下建立一个`.gitignore`文件, 内容如下:

```
 *
 !.gitignore
```

# git tips
## 时光机
Oh shit, I did something terribly wrong, please tell me git has a magic time machine!?!

``` shell
git reflog
# you will see a list of every thing you've done in git, across all branches!
# each one has an index HEAD@{index}
# find the one before you broke everything
git reset HEAD@{index}
# magic time machine
```
## 提交之后, 发现一些小的修改需要一并提交
Oh shit, I committed and immediately realized I need to make one small change!

``` shell
# make your change
git add . # or add individual files
git commit --amend
# follow prompts to change or keep the commit message
# now your last commit contains that change!
```
## 修改最后的提交注释
Oh shit, I need to change the message on my last commit!

``` shell
git commit --amend
# follow prompts to change the commit message
```

## 已经add的文件diff不显示
Oh shit, I tried to run a diff but nothing happened?!
``` shell
git diff --staged
```
Git won't do a diff of files that have been add-ed to your staging area without this flag. File under ¯\_(ツ)_/¯ (yes, this is a feature, not a bug, but it's baffling and non-obvious the first time it happens to you!)



# 乐乐屋同步
1. 开发在lelewu分支
2. 线上在lelewu-product分支

## 更新线上代码
1. git checkout lelewu & git pull
2. git checkout lelewu-product & git merge lelewu
如果有冲突, 需要解决冲突, 如果冲突不重要, 你也可以用`git merge lelewu -X theirs`解决, 如果有其他问题, 直接把线上分支reset `git reset --hard`
3. npm run build:prod
4. python manage.py collectstatic
