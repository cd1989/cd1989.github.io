---
layout: post
title: "你真的懂 Git 吗"
subtitle: ""
date: 2019-05-20
author: "Dylan Chen"
header-img: "img/post-bg-unix-linux.jpg"
tags:
  - Git
published: true
---

相信你一定对 git 并不陌生，日常开发都在用 git 来管理代码，但是你真的懂它吗？不妨思考一下这几个问题：

- 不小心 commit 了一个大文件，删掉后还会占用 git repo 的空间吗？
- git 是否保存的是文件的 diff？
- 删除一个 branch 真正是删除了什么？
- Detached HEAD 是什么意思?
- rebase 和 merge 有什么区别？

如果你对这几个问题还不甚清楚，平时也就使用几个常用的命令，那么不妨跟着这篇文章深入了解一下 git。

# git 定义

如果要给 git 下一个定义，你会怎么说？

通常我们会说 _“git 是一个分布式版本管理系统”_ ，从功能上来说，这个说法没有问题，但是它没有揭露出 git 的本质，所以我喜欢这样去定义 git：

> 本质上来说，git 是一个内容寻址（Content-Addressable）的文件系统，同时在这之上提供了一个版本管理的用户操作界面

什么是内容寻址的文件系统呢？简单说就是通过文件内容生成 key 去访问文件，文件通过内容去定位，而不是通过路径（存储位置）去定位，通常使用 SHA-1 或 MD5 摘要算法去计算文件内容的 key。我们也可以把它看成是一个 key-value 的存储，key 就是文件内容的哈希值，value 就是文件的内容。

git 在这样一个内容寻址的文件系统之上提供了一套命令行操作界面，以方便进行版本管理，如：

- git add
- git commit
- git checkout
- git branch
- ...

# git 实现原理

## 内容寻址文件系统

我们先来看看 git 内容寻址的文件系统是如何实现的。

先初始化一个 git repo：

```bash
$ mkdir test
$ cd test
$ git init
$ tree -L 1 .git
.git
├── HEAD
├── branches
├── config
├── description
├── hooks
├── info
├── objects
└── refs
$ find .git/objects -type f
```

初始化一个空的 git repo，我们看到会生成一些固定的文件和目录，我们先只看 objects 目录，目前该目录是一个空目录。

### 文件（blob）

我们可以通过 git 提供的底层命令 `git hash-object` 插入一个文件到 git 中。其中 `-w` 告诉命令将文件写到 git 中，否则只会计算出文件的哈希 key。可以看到文件的哈希 key 是一个 40 个字符的哈希值，在 `.git/objects` 我们也看到了写入的文件，文件的存储路径把 key 的前两个字符作为了一级目录名称，剩余的 38 字符作为了最终的文件名。

```bash
$ echo "Hello, World~" > /tmp/a.txt
$ git hash-object -w /tmp/a.txt
2dd54fa7a80b680d53d462c10b316bcde2338929
$ find .git/objects -type f
.git/objects/2d/d54fa7a80b680d53d462c10b316bcde2338929
```

既然文件是内容寻址的，我们通过 key 就可以获得文件的内容，我们可以使用 git 提供的 `git cat-file` 来获取文件内容，这里 `-p` 告诉命令根据内容的类型（之后会讲到 git 中不容的内容类型）合适地展示内容。

```bash
$ git cat-file -p 2dd54fa7a80b680d53d462c10b316bcde2338929
Hello, World~
```

也可以使用端的 key 去获取文件内容

```bash
$ git cat-file -p 2dd54fa
Hello, World~
```

让我们修改一下文件内容再写入 git，可以看到 git 里保留了文件的两个版本，可以分别通过 2dd54fa7a80b680d53d462c10b316bcde2338929、2dda938a750b587e8ba7ae7108a499b120c51cf1 确获取。

```bash
$ echo "Hello, Git~" > /tmp/a.txt
$ git hash-object -w /tmp/a.txt
2dda938a750b587e8ba7ae7108a499b120c51cf1
$ find .git/objects -type f
.git/objects/2d/d54fa7a80b680d53d462c10b316bcde2338929
.git/objects/2d/da938a750b587e8ba7ae7108a499b120c51cf1
```

如果这时候你想要把 /tmp/a.txt 回退到之前的版本，可以这样实现：

```bash
$ git cat-file -p 2dd54fa7a80b680d53d462c10b316bcde2338929 > /tmp/a.txt
$ cat /tmp/a.txt
Hello, World~
```

通过这几个简单的命令，我们已经能做到最简单的版本管理，但是去记住哈希 key 太过困难，而且我们存储的数据只有文件的内容，像文件路径等信息并没有记录。这里存储在 `.git/objects` 里的文件对象在 git 里称为 `blob`，后面将介绍其他几种类型的对象，如 `tree`。

### 树 (tree)

一个 `blob` 对象只能对应于一个文件，而且没有文件路径等信息。如果想要管理多个文件，如一个目录，我们就需要使用 `tree` 这种对象了。可以将 `blob` 看成是 Unix 文件系统中的文件，而 `tree` 看成是目录。`tree` 对象包含了一系列指向 `blob` 或其他 `tree`（子树）的哈希 key，这和 Unix 文件系统是类似的。

要创建 `tree`，我们需要通过暂存区（`staging area`）来实现（暂存区也称之为 `index`）。首先需要添加文件到暂存区：

```bash
$ echo "AAAA" > a.txt
$ echo "BBBB" > b.txt
$ git add a.txt
$ git add b.txt
```

有了暂存区后，我们可以创建 `tree`，它将包含暂存区里的两个文件,同时我们看到树里存了文件的属性以及文件路径。

```bash
$ git write-tree
de15c9d22e7bf23908c702aa684d267ade6fe41b
$ git cat-file -p de15c9d22e7bf23908c702aa684d267ade6fe41b
100644 blob 43d5a8ed6ef6c00ff775008633f95787d088285d	a.txt
100644 blob ba629238ca89489f2b350e196ca445e09d8bb834	b.txt
$ git cat-file -t de15c9d22e7bf23908c702aa684d267ade6fe41b
tree
```

上面我们创建了一个 `tree` 对象，它是项目目前的一个快照，目前包含两个文件。让我们修改其中的一个文件，并创建一个新的目录，最后创建新的 `tree` 对象。

```bash
$ echo "aaaa" > a.txt
$ git update-index a.txt
$ mkdir dir1
$ echo "CCCC" > dir1/c.txt
$ git add dir1
$ git write-tree
$ git cat-file -p ac8701b8d9c6e2782fc55057e0f818a4e91ef345
100644 blob 5d308e1d060b0c387d452cf4747f89ecb9935851	a.txt
100644 blob ba629238ca89489f2b350e196ca445e09d8bb834	b.txt
040000 tree c5380f64cfde55f76c8560cdcf642087c405ed02	dir1
$ git cat-file -p 5d308e1d060b0c387d452cf4747f89ecb9935851
aaaa
```

我们看到新的 `tree` 里文件 `a.txt` 已经改变，同时包含了新创建的目录，它是项目最新的快照。

### Commit

不同的 tree 描述了项目不同的快照，但是通过哈希 key 去使用 tree 同样麻烦，另外 tree 只包含了它的内容，却没有谁保存了这个快照、为什么要保存这个快照等信息，这正是 `commit` 对象要解决的问题。

`commit` 对象包含了一个 `tree` 对象的引用，同时包含了一些额外的信息，如作者、描述信息。如果想要创建一个 `commit`，可以使用 `git commit-tree`。下面的例子中我们看到创建的 `commit` 中包含了一个 tree 对象和其他的一些信息。

```bash
$ echo "First commit~" | git commit-tree ac8701b8d9c6e2782fc55057e0f818a4e91ef345
51e1d388dce309de92b0f2fe19476c2a0b4a9642
$ git cat-file -p 51e1d388dce309de92b0f2fe19476c2a0b4a9642
tree ac8701b8d9c6e2782fc55057e0f818a4e91ef345
author cd1989 <chende@caicloud.io> 1558624472 +0800
committer cd1989 <chende@caicloud.io> 1558624472 +0800

First commit~
```

在创建 `commit` 的时候，我们可以指定一个父 commit：

```bash
$ echo "Second commit~" | git commit-tree de15c9d22e7bf23908c702aa684d267ade6fe41b -p 51e1d38
a9a4b12655f3d2b414b7e36be02ca8e6a063176e
```

现在我们创建了两个 commit，并且后一个 commit 指向了前一个 commit，我们可以通过 `git log` 来查看 git 历史：

```bash
$ git log --stat a9a4b12655f3d2b414b7e36be02ca8e6a063176e                                                                                                                128 ↵
commit a9a4b12655f3d2b414b7e36be02ca8e6a063176e
Author: cd1989 <chende@caicloud.io>
Date:   Thu May 23 23:21:31 2019 +0800

    Second commit~

 a.txt      | 2 +-
 dir1/c.txt | 1 -
 2 files changed, 1 insertion(+), 2 deletions(-)

commit 51e1d388dce309de92b0f2fe19476c2a0b4a9642
Author: cd1989 <chende@caicloud.io>
Date:   Thu May 23 23:14:32 2019 +0800

    First commit~

 a.txt      | 1 +
 b.txt      | 1 +
 dir1/c.txt | 1 +
 3 files changed, 3 insertions(+)
```

到目前为止我们已经通过底层的 git 命令创建出了一个完整的 git 历史，而我们平时用的 git 就是基于这一套内容寻址的文件系统构建的。

# Refs

- https://git-scm.com/book/en/v1/Git-Internals
