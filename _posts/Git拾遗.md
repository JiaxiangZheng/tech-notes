## Git 拾遗

![GIT LOGO](https://git-scm.com//images/logo@2x.png)

> 本文大部分命令说明图来自 <https://marklodato.github.io/visual-git-guide/index-zh-cn.html#cherry-pick>

Git 使用已经有很长一段时间了，但大部分情况下用到的只是向个人代码库中的添加文件、提交记录及推送远端等几个简单常用的命令。最近看到了 [githug](https://github.com/Gazler/githug) 项目（好吧，原来这个项目早在四年前就有了），决定在以前写的简单 [Git 入门文章](http://octman.com/blog/2013-2013-08-13-note-git/) 基础上好好重新梳理一下自己的 Git 技能。具体的详细使用方式可以使用 `git command --help` 查看。

### `git mv` & `git grep`

与对应的正常 UNIX 命令类似，`git mv` 用于移动或重命名文件或文件夹，不一样的地方在于 `git mv` 会把修改加入到暂存区，之后直接提交就好了。而 `git grep` 也是类似于 `grep`，用于在 Git 管理的代码中进行快速匹配（也可以使用命令选项设置查找范围为非管理的代码），使用姿势与正常 `grep` 基本一致。

```bash
$ git mv file file-new  #
$ git grep -n 'TODO'   # 找出所有包含 TODO 字符的文件，并显示行号
```

### `git add` & `git commit`

有一种情况是，某个文件的修改内容我们希望分多次提交（可能修改的涉及多个功能点），如果直接用 `git add file && git commit`，我们是无法达到这个目的的。事实上呢，`git add`、`git commit` 是可以只提交部分信息的，只不过我们需要加一个 `-p` 选项。这时候，会出现

    Stage this hunk [y,n,q,a,d,/,e,?]?

使用 `e` 我们就可以编辑选择提交部分内容了。

`git commit` 还允许我们对提交的日期进行「篡改」，例如 `git commit -m "xxx" --date "2015-12-12 00:00:00"`。

还有一个很常见的情形，不得不提。当我们正在紧张的开发时，突然接到一个紧急的 Bug 需要修复，需要新开分支修复，但又不想提交那些开发到一半的功能时，怎么办？

`git stash` 允许我们临时将当前的现场缓存起来，并可以在以后恢复现场。

```bash
$ git stash # 缓存现场
$ git stash list # 显示当前所有的缓存现场列表
$ git stash pop # 恢复现场并删除缓存
```
比如没有提交之前，先 pull 并 rebase 的时候，可以先把本地的修改 stash 起来。

### `git tag` & `git remote`

提交（commit）之后，会自动生成一个哈希值唯一标识这次提交，但有时候我们会希望用一些更有意义的名字来标识一些重要的提交记录（比如版本号等），`git tag` 提供了这个方式。因为 tag 实际上并不会更改提交记录，只是给提交记录又叫了一个名字，本质上是基于指针的，所以基本不会有什么消耗。

```bash
$ git tag v0.1  # 会给最新的 commit 加上 tag
$ git tag v0.1 md5-commit-hash  # 给 md5-commit-hash 打 tag
$ git tag   # 显示所有的 tag 记录
```

有了本地的提交信息之后，需要将最新修改同步到远端服务器上。`git remote` 就是用于远端服务器相关的设置。

```bash
$ git remote -v   # 显示 remote 详细列表信息
$ git remote add origin remote-git-url  # 添加远程 git 地址
```

### `git merge` VS. `git rebase`

开发过程中，难免会切分出很多个分支以便不同功能的处理，这样就很容易出现下面这种情况：

    A---->B---->C---->D   主分支   master
           \
            -----E        功能分支 feature-x

当完成 E 提交后，我们希望合并回主分支，所以会用到

```bash
$ git checkout master && git merge feature-x
```

但这样，会有一个问题，即生成一个新的提交记录 F：

    A---->B---->C---->D-->F   主分支   master
           \            /
            -----E-----        功能分支 feature-x

这样做的后果是，如果协作的人数增多，或者分支数庞大，整个提交记录图就会堪比地铁图，完全无法正常去查看提交记录了。此时，也许需要考虑使用 `git rebase` 命令了。

```bash
$ git checkout master && git rebase feature-x
```

`git rebase` 并不会生成一个新的提交记录，它会在 D 的基础上，将分支提交生成的 patch 依次打上，生成一条线性的提交记录，即：

    A---->B---->C---->D-->E   主分支   master

这样来看，定位问题等就会清晰很多了。但是，`git merge` 也有一定的使用场景，比如开发新功能，我们就是希望生成一个分支提交再合进主分支，这时候建议使用 `git merge --no-ff` 的方式合并分支，以避免出现快照合并。

`git rebase` 还有一个有用的用途，修改历史提交记录，压缩提交记录。比如我们在开发中不小心提交了两次 commit，但考虑再三我们希望这两个 commit 能够合并成一个，毕竟属于同一个问题。这时只需要 `git rebase -i [parent-hash]` 在界面中进行修改即可。

既然这里提到了合并，就不得不提一下 `git cherry-pick` 了。开发了一个功能后，因为种种原因，我们不想它完整地合进开发分支中，但又希望把其中的某几个提交提出来合到开发分支中。这时，简单的 `cherry-pick` 一下，就能实现了。

> 建议：在正常的多人协作开发过程中，我们完成提交并准备 push 的时候，如果出现需要合并远端分支的情况时，更应该使用 `git pull --rebase` 而非单纯的 `git pull` 命令，以生成一条清晰的干净提交记录。

### `git bisect` & `git blame`：定位问题利器

先说一下 `git bisect`，问题场景很简单：当我们最新的代码出现了 regression issue 时，我们需要定位出是哪次提交导致的。有了 `git bisect`，可以以最快的方式切换不同提交场景来定位问题，其基本流程如下：（假定 `v1.0.0` 版本是没有问题的）

```bash
$ git bisect start
$ git bisect bad master
$ git bisect good v1.0.0
```

执行完上述命令后，git 自动切换到 master 和 `v1.0.0` 之间的中间版本，对这个版本进行测试。

```bash
$ git bisect bad    # 如果中间版本有问题，标记为 git bisect bad，然后二分就会在中间版本和 master 之间继续
$ git bisect good   # 如果中间版本没有问题，标记为 git bisect good，然后二分就会在中间版本和 v1.0.0 之间继续
```

另一个有用的命令是 `git blame`，比如某一行代码是有问题的，我们需要找出到底是谁改的，这时候，`git blame file`就可以列举出每一行的提交相关信息了。

```bash
$ git blame -L 10,+4 file
```

**TODO LIST**：

- git merge --squash feature-branch
