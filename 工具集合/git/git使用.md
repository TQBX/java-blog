

[toc]

## Git工作流程

![img](img/git使用/1771072-20201201182439908-1161880852-1609079851160.png)

## Git工作区、暂存区和版本库

- **工作区：**就是你在电脑里能看到的目录。
- **暂存区：**英文叫 stage 或 index。一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
- **版本库：**工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库。

![img](img/git使用/1771072-20201201182448047-584217440.jpg)

- `git add`，暂存区目录树更新，工作区修改【或新增】的文件内容被写入到对象库的一个新对象中，而该对象的ID被记录在暂存区的文件索引中。
- `git commit`，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。
- `git reset HARD`，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
-  `git rm --cached <file>`，直接从暂存区删除文件，工作区则不做出改变。
- `git checkout .`或`git checkout -- <file>`，用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。
- `git checkout HEAD .`或`git checkout HEAD <file>`，用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

## Git基本操作

![img](img/git使用/1771072-20201201182509919-108579028.jpg)

- workspace：工作区
- staging area：暂存区，缓存区
- local repository：本地仓库
- remote repository：远程仓库

基本操作流程：

```shell
git init # 初始化仓库，生成.git目录
git add <file> # 添加文件到暂存区
git commit -m"<message>" # 将暂存区的内容添加到本地仓库中
```

## Git常用命令

### git init

初始化仓库

```shell
git init # 在当前目录下初始化git仓库 
git init newrepo # 在newrepo目录下初始化git仓库
```

在执行完成 git init 命令后，Git 仓库会生成一个 .git 目录，该目录包含了资源的所有元数据，其他的项目目录保持不变。

### git clone

下载项目

```shell
git clone <repo> # 克隆git仓库的项目到当前目录
git clone <repo> <directory> # 克隆git仓库的项目到指定目录
```

### git config

设置git的配置信息

```shell
git config # 显示git的配置信息

# 针对系统上的所有仓库，修改提交代码的用户信息
git config --global user.name "summerday"
git config --global user.email 13279076@qq.com
```

### git add

添加文件到仓库

```shell
git add [file1] [file2] # 添加文件file1和file2进入暂存区
git add [dir] # 添加dir目录进入暂存区，包括子目录
git add .     # 添加当前目录下所有文件到暂存区
```

```shell
$ touch README.md
$ git add README.md
```

### git status

查看在你上次提交之后是否有对文件进行再次修改。

```shell
$ git status

On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   README.md
```

通常使用`-s`获得简短的输出结果。

```shell
$ git status -s

A  README.md  # A 表示新添加
```

### git diff

比较文件在暂存区和工作区的差异，主要应用场景

```shell
git diff [file]  # 尚未缓存的改动
git diff --chached [file]  # 查看已缓存的改动
git diff --stat [file]   #显示摘要而非整个diff
```

前置条件：已经将README.md加入到暂存区。

```shell
vim README.md  # 编辑 modified by summerday

$ git status -s
AM README.md

$ git diff # 显示暂存区和工作区的差异
diff --git a/README.md b/README.md
index e69de29..4dc8994 100644
--- a/README.md
+++ b/README.md
@@ -0,0 +1 @@
+modified by summerday!

$ git diff --cached README.md # 显示暂存区和上一次提交(commit)的差异
diff --git a/README.md b/README.md
new file mode 100644
index 0000000..e69de29

$ git diff --stat README.md  #显示摘要而非整个diff
README.md | 1 +
1 file changed, 1 insertion(+)

```

### git commit

将暂存区内容添加到本地仓库中。

```shell
git commit -m [message]
git commit [file1] [file2] -m [message]
git commit -a
```

```shell
$ git commit -m "first commit !"
[master (root-commit) b81d562] first commit !
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 README.md
 
$ git status
On branch master
nothing to commit, working tree clean
```

`nothing to commit, working tree clean`，表示我们在最近一次提交之后，没有做任何改动，是干净的工作目录。

可以使用`-a`选项跳过`git add`。

```shell
$ vim README.md # add : modified again!

$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")

$ git commit -am "modify and commit again!"
[master 4c3121a] modify and commit again!
 1 file changed, 1 insertion(+)
```

### git reset

回退版本，可以指定退回某一次提交的版本

```shell
git reset [--soft | --mixed | --hard] [HEAD] 
```

`--mixed` 为默认，可以不用带该参数，用于**重置暂存区的文件与上一次的提交(commit)保持一致，工作区文件内容保持不变**。

```shell
git reset [HEAD]
git reset --mixed [HEAD]
```

```shell
$ git reset HEAD # 取消之前git add添加到暂存区的内容
$ git reset HEAD^ # 回退所有内容到上一版本 git reset HEAD~1
$ git reset HEAD^ 1.txt # 回退1.txt文件的版本到上一版本
$ git reset [version] # 回退到指定版本
```

> `--soft`，`--mixed`，`--head`的区别可以对照：[git reset soft,hard,mixed之区别深解](https://www.cnblogs.com/kidsitcn/p/4513297.html)，总结如下：git reset命令是用来将当前branch重置到另外一个commit的，而这个动作可能会将index以及work tree同样影响，而三个参数表示影响程度上的不同：
>
> - `--soft`：仅仅将头指针恢复，【已经add进暂存区的内容以及工作空间的所有东西都不变】。
> - `--mixed`：将头指针恢复，已经add进暂存区的内容也会丢失，【工作空间的代码什么的是不变的，可以根据情况再次进行add，commit操作】。
> - `--hard`：一切就全都恢复了，头指针恢复，已经add进暂存区的内容也会丢失，代码什么的也恢复到以前状态，所以该参数需谨慎使用。

#### HEAD说明

| ^表示  | 数字表示 | 含义       |
| ------ | -------- | ---------- |
| HEAD   | HEAD~0   | 当前版本   |
| HEAD^  | HEAD~1   | 上一版本   |
| HEAD^^ | HEAD~2   | 上上个版本 |

### git rm

删除文件

```shell
git rm <file> # 从暂存区和工作区中删除文件

git rm -f <file> # 如果删除之前修改过，并已经放到暂存区中，需要强制删除
```

## Git Idea 操作

### changelist

一些不需要提交的代码，添加到another changelist

![image-20220615224706915](img/git%E4%BD%BF%E7%94%A8/image-20220615224706915.png)

### new branch

右下角 master， 点击创建新的分支， checkout branch切换到新的分支。

![image-20220615224814070](img/git%E4%BD%BF%E7%94%A8/image-20220615224814070.png)

### 测试分支操作

在master分支上commit之后，显示hyh-dev和master的区别

![image-20220615225115456](img/git%E4%BD%BF%E7%94%A8/image-20220615225115456.png)

切换到 hyh-dev分支，此时可以看到master分支的commit操作， 且此时当前正在hyh-dev分支上。

![image-20220615225306277](img/git%E4%BD%BF%E7%94%A8/image-20220615225306277.png)

并且在刚刚master提交的文件上， 加入不同内容， 以创建冲突， 此时点击update project。

![image-20220615225546034](img/git%E4%BD%BF%E7%94%A8/image-20220615225546034.png)

是没有任何反应的。（有可能是 master只是commit了， 并没有push， 后面再验证）

先commit一下， 可以看到hyh-dev的commit记录。 再push一下。



切换回master分支。也是可以正常push的。



将hyh-dev分支上开发的代码合并到master上。首先切换到master分支上， 选择hyh-dev上的merge hyh-dev into master。

![image-20220615230048633](img/git%E4%BD%BF%E7%94%A8/image-20220615230048633.png)

此时产生冲突，可以选择接受任何一个， 也可以选择更加细致的解决冲突方法 Merge。

![image-20220615230145596](img/git%E4%BD%BF%E7%94%A8/image-20220615230145596.png)

解决冲突即可

![image-20220615230348623](img/git%E4%BD%BF%E7%94%A8/image-20220615230348623.png)

解决完冲突后，此时master处于commit状态， 接着push一下。

![image-20220615230449318](img/git%E4%BD%BF%E7%94%A8/image-20220615230449318.png)

ok，此时master已经提交至远端。而且master分支已经做出了修改。

但是切换至hyh-dev后， 其实还并没有和master提交之后的同步起来。现在的状态是

master的本地和远程都已经是修改后的模样， 但是hyh-dev和远端的hyh-dev都还是上一次push的状态。



如果想要同步成master的模样 ， 就切换到hyh-dev分支， 然后merge master into hyh-dev，由于已经处理过一次冲突， 那么merge没什么问题。

![image-20220615231122104](img/git%E4%BD%BF%E7%94%A8/image-20220615231122104.png)

ok现在是commit状态， 最后将hyh-dev推到远端。

![image-20220615231205586](img/git%E4%BD%BF%E7%94%A8/image-20220615231205586.png)

此时该四个分支都已经是最新的状态了。

