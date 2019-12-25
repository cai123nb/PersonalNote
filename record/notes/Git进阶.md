# Git 进阶学习

Git 进阶学习, 持续补充平时工作学习时遇到的相关 Git 知识.

## Git 文档和推荐教程

- [Git 官方文档](https://git-scm.com/about)
- [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/896043488029600)

## Git 入门

### Git 安装

- 下载地址: [官方下载地址](https://git-scm.com/downloads).
- 选择对应系统的安装包, 点击运行即可.
- 校验是否安装成功, 打开命令行输入`git --version`, 如正确显示版本号即为正确安装.

### Git 基本概念

Git 本地存储分成三大部分:

- Work dir: 工作目录, 实际上操作的目录.
- Index: 暂存区, 临时保存你的更改信息, 需要手动提交.
- History: 版本库, 存储所有的历史提交记录.

每次默认修改是在`Work dir`下, 并不会影响后两个区域的内容. 当我们修改告一段落(完成一个小功能)需要进行存储时, 使用`git add FILE_NAMES`时将修改的内容提交给暂存区. 当我们需要完成修改时, 需要将修改提交给版本库: 首先将需要提交的内容提交给暂存区, 然后通过`git commit -m "COMMENTS_INFO"`将暂存区的内容提交给版本库(注意, 这里提交的是`暂存区`里的内容, 不是`工作目录`的内容.).

### Git 基本操作

- 初始化一个空的 Git 仓库: `git init`, 在本地自动初始化一个新的,空的版本库.
- 提交文件变化到暂存区: `git add FILE_NAME`, 将`FILE_NAME`文件的变更提交给`暂存区`.
- 提交所有文件变化到暂存区: `git add .`, 将`工作目录`中所有文件的变化都提交给`暂存区`.
- 查看差异, `git diff`:
  - 查看工作目录和暂存区文件差异: `git status`, 可以查看到文件级别的差异性. `git diff`, 可以查看到文字级别的差异性.
  - 查看工作目录和版本库中(最近的一次提交)文件的差异(文字级别): `git diff HEAD`.
  - 查看暂存区和版本库(最近的一次提交的)的差异: `git diff --cached`或者`git diff --staged`.
- 提交暂存区里文件到版本库中: `git commit -m "COMMENTS_INFO"`, 将当前暂存区里的所有变化提交给版本库.
- 查看版本库中的提交历史记录: `git log`, 可以查看每次提交的信息.
- 查看用户的操作记录: `git reflog`, 可以看到用户每次在版本库中进行的操作.
- 版本回退或者转移: `git reset VERSION_NUMBER`, 其中`VERSION_NUMBER`可以通过`git log`查看, 或者通过`git reflog`查看. 这里默认转移的方式为`mixed`, 即会将版本转移导致的变化存储到暂存区中, 同时你在本地的修改不会废弃(还存在). 如果想要放弃所有本地更改(强制回到一个一模一样的对应版本), 可以添加`--hard`.
- 废弃工作目录中的文件更改: `git checkout -- FILE_NAME`, 将工作目录中的文件回退到暂存区的版本, 也就是废弃了在提交暂存区这段时间的更改(提示, 如果工作目录**删除**某个文件, 但是已经提交到暂存区中, 使用这个命令也是可以恢复到工作目录的).
- 废弃暂存区中的文件更改: `git reset HEAD FILE_NAME`, 常用于提交前检查文件`git status`, 发现有几个文件不想提交, 但是已经添加到了暂存区. 使用该命令会将文件更改回退到工作目录, 如果工作目录也不想保留这些更改, 只需要使用上一条命令即可(提示, 如果暂存区中不小心使用`git rm`删除了某个文件, 但是版本库中还没删除, 可以使用这个命令恢复到暂存区, 恢复到工作目录, 使用上个命令即可).
- 关联远程仓库:`git remote add REMOTE_NAME xxx.git`, 这里的`REMOTE_NAME`一般会取`origin`(习惯问题).
  - 拷贝远程仓库: `git clone xx.git`.
  - 拉取远程仓库的更新: `git pull REMOTE_NAME REMOTE_BRANCH_NAME`.
  - 向远程仓库推送变化: `git push REMOTE_NAME REMOTE_BRANCH_NAME`, 这里可以给远程仓库推送变化, 注意可能存在冲突(别人已经提交过了, 你的不是最新的), 那就需要拉取下来(上一条), 然后解决冲突, 再次提交.
  - 查看本地关联的所有远程仓库`git remote -v`.
  - 拉取远程分支的所有更新: `git fetch REMOTE_NAME`.
  - 查看某一个远程分支的具体情况: `git remote show REMOTE_NAME`.
  - 修改远程仓库的别名: `git remote rename ORIGIN_REMOTE_NAME CURRENT_REMOTE_NAME`.
  - 删除远程仓库: `git remote rm REMOTE_NAME`.
- 分支情况(branch):
  - 新建分支: `git branch BRANCH_NAME`.
  - 切换分支: `git checkout BRANCH_NAME`.
  - 新建和切换分支: `git checkout -b BRANCH_NAME`.
  - 查看所有分支: `git branch`, 带`*`为当前分支.
  - 查看分支的详细信息(对应提交): `git branch -v`
  - 查看已经合并到当前分支的分支: `git branch --merged`, 对于已经合并的分支(除了`*`)就可以进行删除了.
  - 查看未合并到当前分支的分支: `git branch --no-merged`, 这类分支进行删除会失败,报错. `git`认为这些工作没有保存, 当然可以强制删除`-D`.
  - 合并分支: `git merge MERGE_BRANCH_NAME`, 注意的是, 如果出现了冲突, 就需要修改冲突文件, 然后提交. 默认合并分支时会删除分支信息, 如果想保留可以使用`--no-ff`参数, 如`git merge --no-ff -m "merge with no-ff" dev`.
  - 删除分支: `git branch -d BRANCH_NAME`, 注意, 如果在分支未合并的时候进行删除就会报错, 如果这时候不想保留分支代码, 可以强制删除将`-d`替换为`-D`.
- 保留工作现场: `git stash`, 该命令会将现在工作目录中的更改临时存储, 这时你可以自由地切换分支了.
- 查看工作现场: `git stash list`.
- 回到工作现场: `git stash pop STASH_NAME`或者`git stash apply STASH_NAME`, 两者的区别在于: 前者会主动删除该次工作现场, 后者不会, 必须手动删除`git stash drop STASH_NAME`.
- 本地分支上传到远程仓库: `git push REMOTE_NAME LOCAL_BARANCH_NAME:REMOTE_BRANCH_NAME`, 这样就可以将本地的分支推送到远程.
- 创建远程仓库的分支: `git checkout -b LOCAL_BRANCH_NAME REMOTE_NAME/REMOTE_BRANCH_NAME`或者`git branch --track LOCAL_BRANCH_NAME REMOTE_NAME/REMOTE_BRANCH_NAME`, 这样就会基于远程分支在本地创建对应的分支, 一般推荐两者分支名一致.
- 标签信息, 标签分为两种: `轻量标签`(只是某一次提交的引用)和`附注标签`(含有完整的标签信息, 包括作者, 时间等, 可以进行校验).
  - 创建轻量标签: `git tag TAG_NAME`, 默认给当前提交打上标签.
  - 创建附注标签: `git tag -a TAG_NAME -m "TAG_INFO"`, 默认给当前提交打上标签.
  - 查看标签: `git tag`.
  - 删除标签: `git tag -d TAG_NAME`.
  - 查看特定的标签(正则): `git tag -l 'v1.8.5*'`.
  - 给特定提交打标签: `git tag TAG_NAME COMMIT_VERSION`, 注意这里必须是新的标签, 否则会创建失败, 如果已经用了标签名, 可以先进行删除.
  - 查看标签绑定的记录: `git show TAG_NAME`.
  - 推送某次标签到远程仓库: `git push REMOTE_NAME REMOTE_BRANCH TAG_NAME`, 注意这里的标签名必须提前创建好, 不然会报错.
  - 推送所有标签到远程: `git push REMOTE_NAME REMOTE_BRANCH --tags`.
  - 删除标签, 推送远程更新标签(同步删除记录): `git push origin :refs/tags/DELETE_TAG_NAME`.
  - 切换到标签所在的提交: `git checkout TAG_NAME`, 注意这样会出现`detacthed HEAD`情况, 这时候的提交有些问题, 推荐使用新的分支来进行处理.
  - 切换到标签所在的提交, 并新建分支绑定: `git checkout -b BRANCH_NAME TAG_NAME`.

## 进阶学习

### GIT 追踪远程分支

- 拉取远程分支更新: `git fetch REMOTE_NAME`.
- 本地新建对应分支, 并切换到对应分支: `git checkout -b LOCAL_BRANCH_NAME REMOTE_NAME/REMOTE_BRANCH_NAME`.
- 更改(或者绑定)远程分支: `git branch -u REMOTE_NAME/REMOTE_BRANCH)NAME`, 注意使用这个命令时, 需要切换到对应的分支.
- 本地远程分支更新: `git pull origin _BRANCH_NAME`.
- do something.
- 推送更新到远程分支: `git push origin REMOTE_BRANCH_NAME`.
- 查看本地分支和远程分支的关联情况(追踪情况): `git branch -vv`.
- 删除远程分支: `git push REMOTE_NAME --delete BRANCH_NAME`.

### GIT 变基(Rebase)

我们常见的分支处理, 有: `merge`和`rebase`. 其中`merge`的原理就是, 将两个分支的内容进行比较, 合并, 形成一个新的提交(交合点). 而`rebase`, 变基的处理方式则是: 将另一个分支上的内容, 移动到当前分支(从分叉点开始), 类似于`重新播放`, 然后再将当前分支的内容移动到前面. 这样子看起来就好像只有一条分支(一条直线), `彻底消化`了另一个分支.

使用`rebase`需要遵从一个准则: **不要对在你的仓库外有副本的分支执行变基。**, 如果是自己本地的分支, 你可以随便使用. 但是如果这个分支有其他人使用的话, 那就不要使用变基, 不然使用的那个人需要解决各种冲突.

### GIT误删文件如处理

请教一: 当我们不小心使用`git rm`删除了某个文件, 后悔了想要找回怎么办?

分析: 当我们使用`git rm`时, 我们将文件从我们的工作目录中删除, 然后放置到暂存区中, 这时候还没提交. 我们可以通过废弃暂存区提交, 然后使用废弃工作目录提交来进行回滚.

- `git reset HEAD FILE_NAME`: 将删除操作从暂存区域回滚, 这时候删除操作就只是在本地工作目录中.
- `git checkout FILE_NAME`: 废弃本地工作目录操作(根据暂存区内容).

情景二: 当我们不小心把一个日志文件(或者编译文件, 编辑器的配置文件等等)提交到版本库了, 想要从版本库进行删除, 但是希望保存在本地(本地不要进行删除).

分析: 这时候直接使用`git rm`对应的文件就不行了, 这样会同样删除掉本地的文件. 这时候就需要使用`git rm --cached FILE_NAME`这样就只会删除版本库中的信息, 而不会删除本地的信息. 然后提交到版本库, 再在本地的`gitignore`文件中添加进去即可.

- `git rm --cached FILE_NAME`, 将文件从追踪列表中移除.
- `git commit`.
- 添加到本地的`gitignore`文件中.

### GIT 自定义git log

查看提交记录时, 直接使用`git log`, 可以查看每个提交的 SHA-1 校验和、作者的名字和电子邮件地址、提交时间以及提交说明。

如果想要查看每一次提交, 具体的文件内容变更, 加上`-p`就可以, 如果需要限制查看的数量(n), 添加`-n`参数即可.

还有一个常见的参数就是`--stat`, 会在每次提交中携带一些常见的统计信息.

另外一个常见的格式化参数就是`--pretty`, `pretty`可以和很多命令进行结合, 常见的有: `oneline, full, fuller, short`. 如`--pretty=oneline`, 所有的提交都会以一行的形式展示.

其中一个重要的组合就是`format`, 自定义输出格式, 支持的参数有:

|参数|说明|
|:-|:-|
|%H|提交对象(commit)的完整哈希字串|
|%h|提交对象的简短哈希字串|
|%T|树对象(tree)的完整哈希字串|
|%t|树对象的简短哈希字串|
|%P|父对象(parent)的完整哈希字串|
|%p|父对象的简短哈希字串|
|%an|作者(author)的名字|
|%ae|作者的电子邮件地址|
|%ad|作者修订日期(可以用 --date= 选项定制格式)||
|%ar|作者修订日期，按多久以前的方式显示|
|%cn|提交者(committer)的名字|
|%ce|提交者的电子邮件地址|
|%cd|提交日期|
|%cr|提交日期，按多久以前的方式显示|
|%s|提交说明|

如:

```shell
$ git log --pretty=format:"%h - %an, %ar : %s"
ca82a6d - Scott Chacon, 6 years ago : changed the version number
085bb3b - Scott Chacon, 6 years ago : removed unnecessary test
a11bef0 - Scott Chacon, 6 years ago : first commit
```

另外的一些常见选项有:

|参数|说明|
|:-|:-|
|-p|按补丁格式显示每个更新之间的差异。|
|--stat|显示每次更新的文件修改统计信息。|
|--shortstat|只显示 --stat 中最后的行数修改添加移除统计。|
|--name-only|仅在提交信息后显示已修改的文件清单。|
|--name-status|显示新增、修改、删除的文件清单。|
|--abbrev-commit|仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。|
|--relative-date|使用较短的相对时间显示(比如，“2 weeks ago”)。|
|--graph|显示 ASCII 图形表示的分支合并历史。|
|--pretty|使用其他格式显示历史提交信息。可用的选项包括 oneline，short，full，fuller 和 format(后跟指定格式)。|

另外`git log`还支持按照不同条件来筛选提交记录:

|参数|说明|
|:-|:-|
|-(n)|仅显示最近的 n 条提交|
|--since, --after|仅显示指定时间之后的提交。|
|--until, --before|仅显示指定时间之前的提交。|
|--author|仅显示指定作者相关的提交。|
|--committer|仅显示指定提交者相关的提交。|
|--grep|仅显示含指定关键字的提交|
|-S|仅显示添加或移除了某个关键字的提交|

如: 要查看 Git 仓库中，2008 年 10 月期间，Junio Hamano 提交的但未合并的测试文件，可以用下面的查询命令:

```shell
git log --pretty="%h - %s" --author=gitster --since="2008-10-01" \
  --before="2008-11-01" --no-merges -- t/
```

### GIT 自定义配置

初次使用`GIT`时, 一般需要进行自定义的配置: 配置基本的用户信息(便于提交时辨识度), 配置自定义的别名(方便自己的操作习惯)等. 配置信息分成三层: `系统层`, `用户层`和`当前项目层`, 分别对应的后缀是`--system`, `--global`和无. 一般设置`用户层`即可. 优先级后者大于前者, 如`当前项目层`大于`用户层`.

- 配置用户名: `git config user.name USER_NAME --global`.
- 配置用户邮箱: `git config user.email cjyong@cmbchina.cn --global`.
- 查看所有的配置的信息: `git config --list`.
- 查看具体的配置信息: `git config CONFIG_NAME(如user.email)`.
- 配置别名: `git config --global alias.ALIAS ACTUAL_NAME`, 如: `git config --global alias.st status`使用`git st`来指代`git status`, 自定义显示: `git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"`.

### 更新 fork 仓库

当很多团队对同一个项目进行代码开发(或者参与开源项目开发)时, 开发方式通常为`fork`公共的代码仓库到自己的仓库, 然后本地`clone`下来, 开发修改代码, 推送到自己的仓库代码, 然后在仓库里发起一个`Pull Request`, 当公共代码仓库负责人审查通过之后就会合并到公共代码库中. 当我们`fork`之后, 过了一段时间, 需要开发新功能, 但是这时我们`fork`的项目不是最新的了, 需要基于最新的代码进行开发, 这时候就需要更新我们自己的`fork`代码仓库.

- 配置当前当前 fork 的仓库的原仓库地址: `git remote add upstream OLD_GIT_REP_URL`.
- 查看是否正确建立连接: `git remote -v`.
- 获取原仓库的最新更新: `git fetch upstream`.
- 合并到本地分支。切换到本地 master 分支，合并 upstream/master 分支: `git merge upstream/master`.
- 更新 fork 仓库: `git push origin master`.

### 子模块处理(submodule)

`GIT`通过子模块来处理项目依赖问题, 如果依赖其他项目, 可以通过子模块依赖的方式导入依赖.

- 本地添加子模块依赖: `git submodule add https://xxxxx.git`, 这时候就会拷贝对应的模块信息到本地并生成`.gitmodules`配置文件.
- 查看子模块的文件的变更: `git diff --cached --submodule`.
- 拷贝存在子模块的项目: `git clone --recursive https://xxxx.git`, 这时候会拷贝对应的项目并且会拷贝对应依赖的子模块. 注意, 如果不添加`--recursive`参数就会拷贝一个空的文件夹, 这时候需要使用`git submodule init`进行子模块初始化.
- 对子模块进行远程更新: `git submodule update --remote`. 默认使用 master 分支.
- 更多操作细节,[请查询官方文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

### 更改中间的某次提交

当我们不小心提交了一个错误的提交, 我们需要撤销这次提交怎么办呢? 使用`git revert COMMIT_ID`即可, 这样会在当前的头部生成一个新的提交, 这个提交的内容就是撤销了这次提交的内容, 当然你还可以进行自定义的修改, 然后使用`git commit --amend`附加到这次提交即可. 最后推送到远程就完成了.

### 更新上次提交

[官方解决方案](https://help.github.com/en/articles/changing-a-commit-message)

如果提交尚未推送到远程, 可以通过`git commit --amend`修改提交的注释信息或者追加修改.

- 修改注释信息: `git commit --amend`, 打开编辑页面, 修改注释信息, 选择`:wq`即可.
- 追加修改信息: 首先将需要追加的修改信息, 添加到暂存区, 提交时使用`git commit -amend`, 直接保存退出即可.

如果未提交到远程, 则可以首先通过`reflog`查看具体的操作日志, 然后使用`reset`跳转到对应的错误提交中, 使用`git commit --amend`更新提交信息, 然后使用`cherry-pick`将之后需要保留的提交保存下来.

如果提交到了远程,就需要使用`rebase`命令, 具体操作参照官方解决方案:

- On the command line, navigate to the repository that contains the commit you want to amend.
- Use the git rebase -i HEAD~n command to display a list of the last n commits in your default text editor.

```java
git rebase -i HEAD~3 // Displays a list of the last 3 commits on the current branch
```

The list will look similar to the following:

```java
pick e499d89 Delete CNAME
pick 0c39034 Better README
pick f7fde4a Change the commit message but push the same commit.

// Rebase 9fdb3bd..f7fde4a onto 9fdb3bd
//
// Commands:
// p, pick = use commit
// r, reword = use commit, but edit the commit message
// e, edit = use commit, but stop for amending
// s, squash = use commit, but meld into previous commit
// f, fixup = like "squash", but discard this commit's log message
// x, exec = run command (the rest of the line) using shell
// These lines can be re-ordered; they are executed from top to bottom.
// If you remove a line here THAT COMMIT WILL BE LOST.
// However, if you remove everything, the rebase will be aborted.
// Note that empty commits are commented out
```

- Replace pick with reword before each commit message you want to change.

```java
pick e499d89 Delete CNAME
reword 0c39034 Better README
reword f7fde4a Change the commit message but push the same commit.
```

- Save and close the commit list file.
- In each resulting commit file, type the new commit message, save the file, and close it.
- Force-push the amended commits. `git push --force`(maybe you need pull first).

### cherry-pick

当我们在`prod`分支上想进行某项功能的测试(代码开发在`dev`分支), 这时候我们想要在`prod`分支上合并`dev`上的某一项功能, 但是又不想合并整个`dev`分支(`dev`分支很多功能还没测试完). 这时候就可以使用`cherry-pick`命令: 挑选其他分支的某次提交进行合并. 用法:

- 切换到主分支(主体分支), 使用`git cherry-pick VERSION_NUMBER`, 合并即可, 这里的版本号可以通过`git log`查看到. 注意, 如果出现冲突, 那就需要解决完冲突, 进行提交.
- 保留原提交的基本信息: `git cherry-pick -x VERSION_NUMBER`.
- 选择某个时间段的提交信息, 进行合并: `git cherry_pick START_VERSION_NUMBER^...END_VERSION_NUMBER`, 注意这里的`^...`, 这表示包括左边的起点, 如果去掉`^`, 就不会包括左边的起始点.

## 常见问题

### 拉取失败时 RefusingMerge

这时候一般`git`认为, 两个仓库可能不是同一个仓库(没有相同的`commit`), 这时候可以使用`git pull origin master --allow-unrelated-histories`告诉`git`, 自己已经确认好了.

### 提交时报错 HEADDetached

[参考博文](https://blog.csdn.net/u011240877/article/details/76273335)

`detached`产生的原因: 当你`checkout`到某一个`commit_id`, 并不是分支时, 你在这时候进行提交代码, 就会产生一个匿名的分支. `git`无法识别你这个提交归属那里. 解决方法为: 在当前匿名分支上建立一个临时分支, 然后切换回主分支, 合并临时分支即可.
