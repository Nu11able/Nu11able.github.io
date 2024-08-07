---
title: Git 笔记
date: '2022-02-26 02:03:00'
tags:
 - tools
categories:
 - tools
---

# Git笔记
## 说明
`1、以下举例所用的仓库地址均为：git@github.com:yourname/repo.git`
`2、以下举例所用的仓库地址均为：github_repo`


## 新建仓库
从github中新创建仓库后可以看到下图

- 本地没有代码从0开始
```bash
echo "# test" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main # 修改master分支名字为main
git remote add origin git@github.com:yourname/repo.git
git push -u origin main
```

- 本地已有代码
```bash
git remote add origin git@github.com:yourname/repo.git
git branch -M main # 可以跳过

git push -u origin main
# or
git push -u origin master # 默认存在master分支
```

## git remote
### 列出远程仓库
```bash
git remote
git remote -v # 列出详细信息
```

### 添加/删除远程仓库地址
```bash
git remote add name git@github.com:yourname/repo.git
git remote remove name
```

### 修改仓库名字
```bash
git remote rename oldname newname
```

## git rm
### 删除对某个文件的跟踪
```bash
git rm --cached somefile

# 如果是文件夹
git rm -r --cached somefile
```

## git reset
### 撤销commit
```bash
git reset --soft HEAD^ # 撤销一次commit
git reset --soft HEAD~2 # 撤销两次commit
```
`提交完了之后发现文件名不对，或者说对某个文件做了轻微的改动`
```bash
git add . # 添加所有新的修改到暂存区
git commit --amend
```

## 常见问题

|命令|功能|
|-|-|


### 修复github只有新建仓库时才有小绿点贡献而commit没有
```bash
git filter-branch --env-filter 'GIT_AUTHOR_EMAIL="your_email@email";GIT_AUTHOR_NAME="your_name";'

```
