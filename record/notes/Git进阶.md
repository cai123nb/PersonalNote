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

  - 新建分支: `git branch BRANCH_NAME (COMMIT_ID)`, 注意如果不传递`commit_id`则默认是从当前`HEAD`所在的提交中建立分支.
  - 切换分支: `git checkout BRANCH_NAME`.
  - 新建和切换分支: `git checkout -b BRANCH_NAME`.
  - 查看所有分支: `git branch`, 带`*`为当前分支.
  - 查看分支的详细信息(对应提交): `git branch -v`
  - 查看已经合并到当前分支的分支: `git branch --merged`, 对于已经合并的分支(除了`*`)就可以进行删除了.
  - 查看未合并到当前分支的分支: `git branch --no-merged`, 这类分支进行删除会失败,报错. `git`认为这些工作没有保存, 当然可以强制删除`-D`.
  - 合并分支: `git merge MERGE_BRANCH_NAME`, 注意的是, 如果出现了冲突, 就需要修改冲突文件, 然后提交.

    - 默认合并分支时会删除分支信息, 如果想保留可以使用`--no-ff`参数, 如`git merge --no-ff -m "merge with no-ff" dev`.
    - 如果在合并时想要忽略空白信息, 可以添加参数`-Xignore-all-space 或 -Xignore-space-change`.

  - 删除分支: `git branch -d BRANCH_NAME`, 注意, 如果在分支未合并的时候进行删除就会报错, 如果这时候不想保留分支代码, 可以强制删除将`-d`替换为`-D`.

- 保留工作现场: `git stash`, 该命令会将现在工作目录中的更改临时存储, 这时你可以自由地切换分支了.

  - 查看工作现场: `git stash list`.
  - 回到工作现场: `git stash pop STASH_NAME`或者`git stash apply STASH_NAME`, 两者的区别在于: 前者会主动删除该次工作现场, 后者不会, 必须手动删除`git stash drop STASH_NAME`.
  - 需要注意的是, 如果保留工作现场时, 在工作目录中存在更新, 且在暂存区内也存在更新时, 这时候恢复工作现场(`git stash apply/pop`)时, 会默认将暂存区的内容放到工作目录下(不会自动在帮你放到暂存区, 即所有的变化都是反映在工作目录下). 这时候可以通过`--index`使其动态添加到暂存区(和保存之前一致).
  - 当我们保存工作现场(`git stash`)时, 如果只想保留工作目录的变化, 不想保留暂存区的变化(默认两个都会保存), 这时候可以通过添加`--keep-index`命令.
  - 当我们保留工作现场(`git stash`)时, 默认只保存已经在索引中(版本库中)的文件, 对于新增的文件, 是不会处理的. 这时候可以添加`-u或者--include-untracked`来进行处理.
  - 如果我们需要对暂存的工作现场重新工作, 但是不想影响到主仓库. 可以新建一个分支进行绑定处理, `git stash branch BRANCH_NAME STASH_NAME`, 就会对对应的工作现场新建一个分支进行处理(注这里会默认切换分支).
  - 与清理相对应的就是清理: `git clean`, 用以移除工作目录中未追踪的文件列表. 如清除所有未追踪的文件和文件夹(及其子目录): `git clean -d -f`. 这是一个高危操作, 推荐在清理之前, 通过`-n`来确认是否清理正确的文件. 默认不会清除`gitignore`中的文件, 如果想要一并清除添加`-x`参数. 同样推荐使用交互模式保证清理文件的正确性`-i`.

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

参数  | 说明
:-- | :-------------------------
%H  | 提交对象(commit)的完整哈希字串
%h  | 提交对象的简短哈希字串
%T  | 树对象(tree)的完整哈希字串
%t  | 树对象的简短哈希字串
%P  | 父对象(parent)的完整哈希字串
%p  | 父对象的简短哈希字串
%an | 作者(author)的名字
%ae | 作者的电子邮件地址
%ad | 作者修订日期(可以用 --date= 选项定制格式) |
%ar | 作者修订日期，按多久以前的方式显示
%cn | 提交者(committer)的名字
%ce | 提交者的电子邮件地址
%cd | 提交日期
%cr | 提交日期，按多久以前的方式显示
%s  | 提交说明

如:

```shell
$ git log --pretty=format:"%h - %an, %ar : %s"
ca82a6d - Scott Chacon, 6 years ago : changed the version number
085bb3b - Scott Chacon, 6 years ago : removed unnecessary test
a11bef0 - Scott Chacon, 6 years ago : first commit
```

另外的一些常见选项有:

参数              | 说明
:-------------- | :-----------------------------------------------------------------
-p              | 按补丁格式显示每个更新之间的差异。
--stat          | 显示每次更新的文件修改统计信息。
--shortstat     | 只显示 --stat 中最后的行数修改添加移除统计。
--name-only     | 仅在提交信息后显示已修改的文件清单。
--name-status   | 显示新增、修改、删除的文件清单。
--abbrev-commit | 仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。
--relative-date | 使用较短的相对时间显示(比如，"2 weeks ago")。
--graph         | 显示 ASCII 图形表示的分支合并历史。
--pretty        | 使用其他格式显示历史提交信息。可用的选项包括 oneline，short，full，fuller 和 format(后跟指定格式)。

另外`git log`还支持按照不同条件来筛选提交记录:

参数                | 说明
:---------------- | :----------------
-(n)              | 仅显示最近的 n 条提交
--since, --after  | 仅显示指定时间之后的提交。
--until, --before | 仅显示指定时间之前的提交。
--author          | 仅显示指定作者相关的提交。
--committer       | 仅显示指定提交者相关的提交。
--grep            | 仅显示含指定关键字的提交
-S                | 仅显示添加或移除了某个关键字的提交

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
- 更新配置(包括别名): `git config --global --edit`.

### 更新 fork 仓库

当很多团队对同一个项目进行代码开发(或者参与开源项目开发)时, 开发方式通常为`fork`公共的代码仓库到自己的仓库, 然后本地`clone`下来, 开发修改代码, 推送到自己的仓库代码, 然后在仓库里发起一个`Pull Request`, 当公共代码仓库负责人审查通过之后就会合并到公共代码库中. 当我们`fork`之后, 过了一段时间, 需要开发新功能, 但是这时我们`fork`的项目不是最新的了, 需要基于最新的代码进行开发, 这时候就需要更新我们自己的`fork`代码仓库.

- 配置当前当前 fork 的仓库的原仓库地址: `git remote add upstream OLD_GIT_REP_URL`.
- 查看是否正确建立连接: `git remote -v`.
- 获取原仓库的最新更新: `git fetch upstream`.
- 合并到本地分支。切换到本地 master 分支，合并 upstream/master 分支: `git merge upstream/master`.
- 更新 fork 仓库: `git push origin master`.

### Reset和Checkout

理解两者的区别时,先明白`HEAD`,`暂存区`和`工作目录`的区别. 其中后两者和上面的定义一样, 这里简单的复述了`HEAD`的定义: 当前分支所引用的指针(总是指向该分支的最后一次分支), 指向版本库树中的一个commit的指针. 如果想要查看具体指向的指针地址: `git cat-file -p HEAD`. 也可以查看当前版本库中每个文件当前版本的哈希值: `git ls-tree -r HEAD`.

`commit`级别中的`reset`的处理步骤(`git reset COMMIT_ID`):

1. 修改`HEAD`指针的位置, 移动到对应`COMMIT_ID`的位置. 等同于`git reset --soft COMMIT_ID`(注意如果`HEAD`指向的是`Master`, 那么`Master`也会进行修改).
2. 将暂存区的内容重置为对应`COMMIT_ID`的版本库中内容. 等同于`git reset --mixed COMMIT_ID`. 注意, `--mixed`是可选的, 默认就是这个级别.
3. 将工作目录的内容重置为对应`COMMIT_ID`版本库中的内容. 等同于`git reset --hard COMMIT_ID`. 注意这是非常危险的, 会覆盖本地的内容.(如果已经提交到版本库, 可以通过`git reflog`找回).

文件级别中的`reset`的处理步骤(`git reset [COMMIT_ID] FILE_NAME`), 注`COMMIT_ID`可选, 不填默认是`HEAD`, 即当前指向的版本库中的版本:

1. 跳过修改`HEAD`的指针位置.(不进行处理)
2. 修改暂存区中对应的文件, 使其和对应版本的文件内容相同. (结束).

即通过`reset`之后, 默认只会修改版本库中的文件, 不会修改`HEAD`和`工作目录`中的内容. 这时候通过`git status`就可以很明显的看出这个差别.

通过上面的上面的原理可以完成一些特殊的操作, 如**合并提交**. 假设版本库内容为`v1 - v2 - v3`当前HEAD所在的位置为`v3`. 现在想要合并`v2`和`v3`.

1. `git reset --soft v1`, 将`HEAD`切换到`v1`的位置.
2. `git commit` 完成. 这时候暂存区和本地工作目录的内容还是`v3`的版本, 直接提交就可以直接从`v1`直达`v3`. 也就完成了**合并提交**的功能.

`commit`级别中的`checkout`处理步骤(`git checkout BRANCH/COMMIT_ID`):

1. 修改`HEAD`指针的位置, 移动到对应`COMMIT_ID`的位置.
2. 将暂存区的内容重置为对应`COMMIT_ID`的版本库中内容.
3. 将工作目录的内容和对应`COMMIT_ID`版本库中的内容进行合并, 注意不是重置(而是合并).

处理的逻辑非常类似`git reset --hard COMMIT_ID`. 但是有一点不同的是, 工作目录的内容会进行简单的合并, 而不是强制重置. 另一个区别的是, `reset`会移动`HEAD`分支的指向, 而`checkout`只会移动`HEAD`自身到另一个分支, 一般不推荐切换到另一个提交里面.

文件级别中的`checkout`处理步骤, `git checkout COMMIT_ID FILENAME`处理结果等同于(`git reset --hard [branch] file`). 这样对工作目录不安全, 会覆盖工作目录的内容(虽然`HEAD`指针不会变化). 如果不添加`COMMIT_ID`, 就默认废弃本地更新, 而是替换成暂存区的内容.

总结:

summary                  | HEAD | Index | Workdir | WD Safe?
:----------------------- | :--- | :---- | :------ | :-------
Commit Level             | -    | -     | -       | -
reset --soft [commit]    | REF  | NO    | NO      | YES
reset [commit]           | REF  | YES   | NO      | YES
reset --hard [commit]    | REF  | YES   | YES     | NO
checkout [commit]        | HEAD | YES   | YES     | YES
File Level               | -    | -     | -       | -
reset (commit) [file]    | NO   | YES   | NO      | YES
checkout (commit) [file] | NO   | YES   | YES     | NO

checkout file 会同时修改index和worid dir吗.

### 子模块处理(submodule)

`GIT`通过子模块来处理项目依赖问题, 如果依赖其他项目, 可以通过子模块依赖的方式导入依赖.

- 本地添加子模块依赖: `git submodule add https://xxxxx.git`, 这时候就会拷贝对应的模块信息到本地并生成`.gitmodules`配置文件.
- 查看子模块的文件的变更: `git diff --cached --submodule`.
- 拷贝存在子模块的项目: `git clone --recursive https://xxxx.git`, 这时候会拷贝对应的项目并且会拷贝对应依赖的子模块. 注意, 如果不添加`--recursive`参数就会拷贝一个空的文件夹, 这时候需要使用`git submodule init`进行子模块初始化.
- 对子模块进行远程更新: `git submodule update --remote`. 默认使用 master 分支. 更新追踪的远程分支名称:`git config -f .gitmodules submodule.SUBMODULE_NAME.branch BRANCH_NAME`. 这里会将新的分支信息写入配置中, 如果推送到远端会影响其他人(如果你只想自己用).
- 设置`git status`默认显示子模块的信息, `git config status.submodulesummary 1`.
- 当我们在子模块上进行工作时, 我们需要在子模块下建立一个对应的分支, 不然直接使用`git submodule update`会让子模块处于`游离的状态`.

  - 首先进入子模块, 检出一个分支对应: `git checkout BRNACH_NAME`.
  - 拉取远程的更新: `git submodule update --morete --merge`, 进行合并.
  - 进行修改, 然后提交. 注意这里还没有推送, 这时候的`submodule`只对你一个人可见,有效.
  - 拉取远程进行更新: `git submodule update --morete --rebase`.
  - 推送到远程. 注意这一步一定要执行, 不然本地主仓库管理的子模块版本, 对于其他人是拉取不到的(只在你的本地).

- 更多操作细节,[请查询官方文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

为了防止在我们提交主仓库代码时, 本地子模块更新没有推送上去的情况(这时候会导致其他人无法拉取的子模块的改动). `git`提供了`check`的配置: `--recurse-submodules`, 如果值为`check`则会在每一次提交的时候检查一遍子模块本地更新是否提交, 如果存在未提交的时候, 自动失败. `git push --recurse-submodules=check`. 另个参数则为`on-demand`, 如果发现存在本地子模块更新没有推送的情况, 会尝试帮你推送子模块, 如果成功就继续推送主仓库代码, 否则失败. 推荐在子模块开发时, 设置`push`默认配置: `git config push.recurseSubmodules check --global`, 这样就不用每次都输入额外的命令(`--recurse-submodules=check`), 也不用担心忘记输入额外的命令了.

为了方便多个子模块的统一操作, `git`提供了`foreach`命令符, 用于批量操作子模块:

```shell
git submodule foreach 'git stash' // 批量保存子模块的进度
git submodule foreach 'git checkout -b featureA' // 在所有子模块中新建分支并切换
git diff; git submodule foreach 'git diff' // 查看项目内所有变化, 包括所有子模块内容变化
```

### 离线神器打包

`Git`可以将它的数据"打包"到一个文件中。 这在许多场景中都很有用。 有可能你的网络中断了，但你又希望将你的提交传给你的合作者们。 可能你不在办公网中并且出于安全考虑没有给你接入内网的权限。 可能你的无线、有线网卡坏掉了。 可能你现在没有共享服务器的权限，你又希望通过邮件将更新发送给别人，却不希望通过`format-patch`的方式传输 40 个提交。

这些情况下`git bundle`就会很有用。`bundle`命令会将`git push`命令所传输的所有内容打包成一个二进制文件，你可以将这个文件通过邮件或者闪存传给其他人，然后解包到其他的仓库中。

假设你有一个项目想要分享给别人: `git bundle create repo.bundle HEAD master`, 这里就会将当前仓库的`master`分支的数据打包成`repo.bundle`文件. 你可以通过`U盘`或者`邮件`发送给别人.

当你接收了一个这个`repo.bundle`文件, 你需要解压在本地然后进行开发: `git clone repo.bundle repo`, 将对应的文件在`repo`文件夹中解析成一个`git`仓库.

这时候她在这个仓库中开发了一部分代码, 提交了三个提交. 我们需要拷贝回去. 如当前的提交记录为:`xxx... - 8723 - 92a3(你的) - 89a1(你的) - 782c(你的)`. 我们需要将我们的工作打包拷贝给别人: `git bundle create commits.bundle master ^8723`, 这样会将她的提交打包成`commits.bundle`文件. 注意这里起点需要选取最后的一个父节点(不要全是你的), 否则`git`无法校验是否是同一个源头.

当你收到这样的一个`commits.bundle`时, 可以先进行校验是否是合法的`git`包(是否是同一个祖先): `git bundle verify ../commits.bundle`, 然后将内容取出并添加到另一个分支里面: `git fetch ../commits.bundle master:other-master`, 这样就会将文件包里的内容, 负载在`other-master`分支上.

### 更改中间的某次提交

当我们不小心提交了一个错误的提交, 我们需要撤销这次提交怎么办呢? 使用`git revert COMMIT_ID`即可, 这样会在当前的头部生成一个新的提交, 这个提交的内容就是撤销了这次提交的内容, 当然你还可以进行自定义的修改, 然后使用`git commit --amend`附加到这次提交即可. 最后推送到远程就完成了.

### 重写提交历史

重写提交历史会修改提交的`SHA-1`的校验和, 不推荐在已提交的记录中进行修改, 一般用于本地使用.

修改提交信息, 通过`git commit --amend`修改提交的注释信息或者追加修改, [参照官方解决方案](https://help.github.com/en/articles/changing-a-commit-message)

这里的修改包括修改评论, 添加一些文件, 移除文件等等. 如给上次提交新增一个文件(`git add`)或者移除一个文件(`git rm`), 然后通过`git commit --amend`追加补丁即可.

如果需要修改多个提交信息或者上上(或者很多次)次的提交信息. 这时候就必须使用`rebase`来完成.注意这里只推荐在本地使用, 不要对已经提交的commit进行变基.

- 确定需要更改的提交是第几个. 可以通过`git reflog`或者`git log -g`来确认. 如确定要修改的是第三个.
- 进入`rebase`配置页面: `git rebase -i HEAD~3`, 注意这里的`-i`是指以交互的形式处理. 这时候页面应该如下打开:

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

- 选中你需要更改的那个提交(注意这里的顺序是倒序的), 将`pick`修改为`edit`或者`reword`可以参照下面的命令解释. 保存退出.
- 然后按照提示, 更新文件(如果需要的话), 添加到暂存区, 使用`git commit --amend`修改提交信息(参照修改上一次的提交的操作方法).(如果选择了多个提交更新, 就会重复这个过程).
- 使用命令`git rebase --continue`, 完成处理.

#### 拓展

通过`rebase`还可以完成很多功能, 如**移除提交**:

```java
pick e499d89 Delete CNAME
pick 0c39034 Better README
pick f7fde4a Change the commit message but push the same commit.
```

修改为:

```java
pick e499d89 Delete CNAME
pick 0c39034 Better README
```

**修改提交的排序**:

```java
pick e499d89 Delete CNAME
pick 0c39034 Better README
pick f7fde4a Change the commit message but push the same commit.
```

修改为:

```java
pick e499d89 Delete CNAME
pick f7fde4a Change the commit message but push the same commit.
pick 0c39034 Better README
```

**压缩提交**:

```java
pick e499d89 Delete CNAME
pick 0c39034 Better README
pick f7fde4a Change the commit message but push the same commit.
```

修改为:

```java
pick e499d89 Delete CNAME
squash f7fde4a Change the commit message but push the same commit.
squash 0c39034 Better README
```

**拆分提交**:

如将提交: `310154e updated README formatting and added blame`拆分为`updated README formatting`和`added blame`.

1. 进入配置页面`git rebase -i HEAD~2`, 修改为`edit`.

  ```java
  edit 310154e updated README formatting and added blame
  pick e499d89 Delete CNAME
  ```

2. 进入前一个提交`git reset HEAD^`(注意这里是第二个, 按照具体情况来进入). 现在就在需要更新的提交中了.

3. 提交第一个文件: `git add xx` -> `git commit -m 'do something'`.

4. 提交第二个文件: `git add xx` -> `git commit -m 'do something'`.

5. 进行变基: `git rebase --continue`.

### 核武器选项: `filter-branch`

批量修改历史记录, 如修改全局的邮箱地址或从每一个提交中移除一个文件, 这时候可以使用`filter-branch`. 推荐在一个测试分支上处理, 如果处理结果符合你的要求, 再硬重置`master`分支.

#### 每一个提交中移除文件

如有人不小心添加了一个密码文件, 而你需要开源项目, 需要在整个历史记录中擦除该文件的提交历史.

```java
git filter-branch --tree-filter 'rm -f passwords.txt' HEAD
```

#### 使用子目录作为新的根目录

当你导入一个项目时, 项目文件夹有很多, 如`trunk`,`tags`,`.back`等等. 但是实际上有用的就是`trunk`, 你不想关注其他文件夹的提交.

```java
git filter-branch --subdirectory-filter trunk HEAD
```

#### 全局修改邮箱地址

你开始工作时忘记运行`git config`来设置你的名字与邮箱地址，或者你想要开源一个项目并且修改所有你的工作邮箱地址为你的个人邮箱地址。任何情形下，你也可以通过`filter-branch`来一次性修改多个提交中的邮箱地址。 需要小心的是只修改你自己的邮箱地址，所以你使用`--commit-filter`.

```shell
$git filter-branch --commit-filter '
   if [ "$GIT_AUTHOR_EMAIL" = "schacon@localhost" ];
   then
         GIT_AUTHOR_NAME="Scott Chacon";
         GIT_AUTHOR_EMAIL="schacon@example.com";
         git commit-tree "$@";
   else
   fi' HEAD
```

如果多次使用`filter-branch`出现报错:

```shell
A previous backup already exists in refs/original/
Force overwriting the backup with -f
```

这里的备份是上一次`filter-branch`的处理备份, 如果你确认上一次的处理结果, 可以使用以下命令清除备份:

```shell
git update-ref -d refs/original/refs/heads/master
```

### cherry-pick

当我们在`prod`分支上想进行某项功能的测试(代码开发在`dev`分支), 这时候我们想要在`prod`分支上合并`dev`上的某一项功能, 但是又不想合并整个`dev`分支(`dev`分支很多功能还没测试完). 这时候就可以使用`cherry-pick`命令: 挑选其他分支的某次提交进行合并. 用法:

- 切换到主分支(主体分支), 使用`git cherry-pick VERSION_NUMBER`, 合并即可, 这里的版本号可以通过`git log`查看到. 注意, 如果出现冲突, 那就需要解决完冲突, 进行提交.
- 保留原提交的基本信息: `git cherry-pick -x VERSION_NUMBER`.
- 选择某个时间段的提交信息, 进行合并: `git cherry_pick START_VERSION_NUMBER^...END_VERSION_NUMBER`, 注意这里的`^...`, 这表示包括左边的起点, 如果去掉`^`, 就不会包括左边的起始点.

### 二分查找

假设你刚刚在线上环境部署了你的代码，接着收到一些 bug 反馈，但这些 bug 在你之前的开发环境里没有出现 过，这让你百思不得其解。 你重新查看了你的代码，发现这个问题是可以被重现的，但是你不知道哪里出了问 题。你可以用二分法`git bisect`来找到这个问题。 假设上个版本是`v1.0`是正常的, 而当前的版本`v2.0`是不正常的. 提交记录为: `v1.0 - 982a3 - 89c34 - a8489c - 8923cc - v2.0`.

1. 启动二分查找, `git bisect start`.
2. 定义当前状态为错误状态: `git bisect bad`.
3. 定义当前已知状态为好的提交点: `git bisect good v1.0`. 确定了提交的范围, 这时候`git`会自动跳转到中间的提交点. 你这时候需要校验一下, 当前提交是不是正常的.
4. 校验当前提交点是否正常: `git bisect RESULT`, 注意`result`为`good`或者`bad`. `git`会依据当前提交点, 重新定位范围, 自动跳转到中间提交点.
5. 重复上面的步骤, 直到只剩下一个提交.
6. 确定了提交问题, 重置二分查找: `git bisect reset`, 重新回到最开始的位置.

当然还有一个更懒的方法, 使用脚本自动执行, 在脚本中定义正常和不正常的条件:

```shell
$git bisect start HEAD v1.0
$git bisect run test-error.sh
```

## 常见问题

### 拉取失败时 RefusingMerge

这时候一般`git`认为, 两个仓库可能不是同一个仓库(没有相同的`commit`), 这时候可以使用`git pull origin master --allow-unrelated-histories`告诉`git`, 自己已经确认好了.

### 提交时报错 HEADDetached

[参考博文](https://blog.csdn.net/u011240877/article/details/76273335)

`detached`产生的原因: 当你`checkout`到某一个`commit_id`, 并不是分支时, 你在这时候进行提交代码, 就会产生一个匿名的分支. `git`无法识别你这个提交归属那里. 解决方法为: 在当前匿名分支上建立一个临时分支, 然后切换回主分支, 合并临时分支即可.
