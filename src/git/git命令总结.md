---
title: git命令总结
tags: 作者:汪帅
grammar_cjkRuby: true
---


1、git文件三种状态：已提交（committed）、已修改（modified）和已暂存
已提交表示数据已经安全的保存在本地数据库中。 
已修改表示修改了文件，但还没保存到数据库中。 
已暂存表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

2、git项目三个工作区域：Git仓库、工作目录以及暂存区域
Git 仓库目录是 Git 用来保存项目的元数据和对象数据库的地方。 这是 Git 中最重要的部分，从其它计算机克隆仓库时，拷贝的就是这里的数据。

工作目录是对项目的某个版本独立提取出来的内容。 这些从 Git 仓库的压缩数据库中提取出来的文件，放在磁盘上供你使用或修改。

基本的 Git 工作流程如下：

在工作目录中修改文件。

暂存文件，将文件的快照放入暂存区域。

提交更新，找到暂存区域的文件，将快照永久性存储到 Git 仓库目录。

如果 Git 目录中保存着的特定版本文件，就属于已提交状态。 如果作了修改并已放入暂存区域，就属于已暂存状态。 如果自上次取出后，作了修改但还没有放到暂存区域，就是已修改状态。

暂存区域是一个文件，保存了下次将提交的文件列表信息，一般在 Git 仓库目录中。 有时候也被称作‘索引’'，不过一般说法还是叫暂存区域。
3、git init
git config --global  user.name "John Doe"
git config --global user.email  johndoe@example.com

再次强调，如果使用了 --global 选项，那么该命令只需要运行一次，因为之后无论你在该系统上做任何事情， Git 都会使用那些信息。 当你想针对特定项目使用不同的用户名称与邮件地址时，可以在那个项目目录下运行没有 --global 选项的命令来配置。

git config --list   所有的属性
git config user.name  查看某一下的配置

git  add *.c    添加文件
git add LICENSE  添加文件
git commit -m 'initial project version'   commit文件

git clone  http://192.168.102.73:8081/yupan/gittestrename.git   从远程仓库复制项目

git clone http://192.168.102.73:8081/yupan/gittestrename.git  mm 从远程仓库复制项目并将项目名称修改为mm

4、工作目录中的文件的状态已跟踪和未跟踪
git status 查看文件处于什么状态

git add 开始跟踪一个文件   这个命令的功能理解为“添加内容到下一次提交中”

只要在 Changes to be committed 这行下面的，就说明是已暂存状态。 如果此时提交，那么该文件此时此刻的版本将被留存在历史记录中。 你可能会想起之前我们使用 git init 后就运行了 git add (files) 命令，开始跟踪当前目录下的文件。 git add 命令使用文件或目录的路径作为参数；如果参数是目录的路径，该命令将递归地跟踪该目录下的所有文件。

修改之后的文件  没有放入暂存区中，也用  git add .gitignore

git status -s  状态简览
M  lib/simplegit.rb
MM .gitignore
A  README.txt

A表示--新添加到暂存区中的文件前面有 A 标记
M表示--修改过的文件
MM表示 --，出现在右边的 M 表示该文件被修改了但是还没放入暂存区，出现在靠左边的 M 表示该文件被修改了并放入了暂存区


5、忽略文件	.gitignore
文件 .gitignore 的格式规范如下：

所有空行或者以 ＃ 开头的行都会被 Git 忽略。

可以使用标准的 glob 模式匹配。

匹配模式可以以（/）开头防止递归。

匹配模式可以以（/）结尾指定目录。

要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（!）取反。

所谓的 glob 模式是指 shell 所使用的简化了的正则表达式。 星号（*）匹配零个或多个任意字符；[abc] 匹配任何一个列在方括号中的字符（这个例子要么匹配一个 a，要么匹配一个 b，要么匹配一个 c）；问号（?）只匹配一个任意字符；如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配（比如 [0-9] 表示匹配所有 0 到 9 的数字）。 使用两个星号（*) 表示匹配任意中间目录，比如`a/**/z` 可以匹配 a/z, a/b/z 或 `a/b/c/z`等。

6、查看已暂存和未暂存的修改
git diff  --查看未暂存的修改
git diff --cached 或者 git diff --staged  查看已暂存的修改

7、提交更新
git commit  是打开一个编辑器  来编辑内容
git commit -m "hello git"   提交的是放在暂存区域的快照，

跳过使用暂存区域
git commit -a -m “hello world”

8、移除文件
git rm tt.txt 
git rm -f tt.txt  
下一次提交时，该文件就不再纳入版本管理了。 如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除选项 -f（译注：即 force 的首字母）。 这是一种安全特性，用于防止误删还没有添加到快照的数据，这样的数据不能被 Git 恢复。
git rm --cached tt.txt 
我们想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。 换句话说，你想让文件保留在磁盘，但是并不想让 Git 继续跟踪。 当你忘记添加 .gitignore 文件，不小心把一个很大的日志文件或一堆 .a 这样的编译生成文件添加到暂存区时，这一做法尤其有用。 为达到这一目的，使用 --cached 选项

git rm log/\*.log
注意到星号 * 之前的反斜杠 \， 因为 Git 有它自己的文件模式扩展匹配方式，所以我们不用 shell 来帮忙展开。 此命令删除 log/ 目录下扩展名为 .log 的所有文件。 类似的比如：

git rm \*~
该命令为删除以 ~ 结尾的所有文件

9、移动文件

git mv README.txt

10、查看提交历史
git log
默认不用任何参数的话，git log 会按提交时间列出所有的更新，最近的更新排在最上面。 正如你所看到的，这个命令会列出每个提交的 SHA-1 校验和、作者的名字和电子邮件地址、提交时间以及提交说明。

git log -p -2 
 -p，用来显示每次提交的内容差异。 你也可以加上 -2 来仅显示最近两次提交：

git log --stat
你想看到每次提交的简略的统计信息,每次提交的下面列出所有被修改过的文件、有多少文件被修改了以及被修改过的文件的哪些行被移除或是添加了

git log --pretty=online

另外一个常用的选项是 --pretty。 这个选项可以指定使用不同于默认格式的方式展示提交历史。 这个选项有一些内建的子选项供你使用。 比如用 oneline 将每个提交放在一行显示，查看的提交数很大时非常有用。 另外还有 short，full 和 fuller 可以用，展示的信息或多或少有些不同，请自己动手实践一下看看效果如何。

$ git log --pretty=oneline
ca82a6dff817ec66f44342007202690a93763949 changed the version number
085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7 removed unnecessary test
a11bef06a3f659402fe7563abf99ad00de2209e6 first commit
但最有意思的是 format，可以定制要显示的记录格式。 这样的输出对后期提取分析格外有用?—?因为你知道输出的格式不会随着 Git 的更新而发生改变：



11、撤销操作
git reset HEAD tt.txt  从暂存中移除
git checkout -- tt.txt


12、远程仓库

git remote -v  会显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL
 
git remote add pb http://192.168.102.73:8081/yupan/gittestrename.git  添加一个新的远程 Git 仓库，同时指定一个你可以轻松引用的简写：

git fetch pb master   拉去远程仓库的信息

git push origin master  推送到远程仓库

git remote show origin  查看远程仓库信息  

git remote rename pb paul 修改远程仓库简写名

git remote rm paul   移除

13、打标签

git tag 查看标签

git tag -l  'v1*'  查看某个系列的标签

git tag  -a v1.7 -m '1.7'  创建标签

git show v1.7  查看某个标签

git log --pretty=oneline
git tag -a v1.2 9fceb02   对过去的提交打标签

git  push origin v1.7 将v1.7标签推送到远程服务器上

git push origin --tags   将会把所有不在远程仓库服务器上的标签全部传送到那里。

git checkout -b  version2  v2.0.0  将标签v2.0.0 检出到分支version2上


1、创建分支
git branch testing  创建一个分支

git log --oneline --decorate   查看各个分支当前所指的对象

git checkout testing  分支切换

git log --oneline --decorate --graph --all ，它会输出你的提交历史、各个分支的指向以及项目的分支分叉情况

git checkout -b iss53   相当于 git branch iss53 和git checkout iss53

git checkout -b hotfix   创建并切换到hotfix分支

git checkout master  切换回 master
git merge hotfix    master合并hotfix分支
合并操作没有需要解决的分歧--这是快进“fast-forward”  此时指针向前推进

你的开发历史从一个更早的地方开始分叉开来（diverged）。 因为，master 分支所在提交并不是 iss53 分支所在提交的直接祖先，Git 不得不做一些额外的工作。 出现这种情况的时候，Git 会使用两个分支的末端所指的快照（C4 和 C5）以及这两个分支的工作祖先（C2），做一个简单的三方合并。

和之前将分支指针向前推进所不同的是，Git 将此次三方合并的结果做了一个新的快照并且自动创建一个新的提交指向它。 这个被称作一次合并提交，它的特别之处在于他有不止一个父提交。

Git 会自行决定选取哪一个提交作为最优的共同祖先，并以此作为合并的基础

 如果你在两个不同的分支中，对同一个文件的同一个部分进行了不同的修改，Git 就没法干净的合并它们。 如果你对 #53 问题的修改和有关 hotfix 的修改都涉及到同一个文件的同一处，在合并它们的时候就会产生合并冲突：
此时 Git 做了合并，但是没有自动地创建一个新的合并提交。 Git 会暂停下来，等待你去解决合并产生的冲突。 你可以在合并冲突后的任意时刻使用 git status 命令来查看那些因包含合并冲突而处于未合并（unmerged）状态的文件：

git branch  -d  hotfix  删除分支

git branch  列出当前所有的分支

git branch -v 查看每一个分支的最后一次提交

git branch --merged  列出已经合并到当前分支的分支
git branch --no-merged 列出没有合并到当前分支的分支

git branch -D hotfix  强制删除分支


2、远程分支
git ls-remote 显示远程引用完整列表
git remote show origin master


git remote add   添加一个新的远程仓库引用

git push origin serverfix:awesomebranch   将本地的serverfix分支推送到远程仓库上的awesomebranch分支

git checkout  -b sf  origin/serverfix  如果想要将本地分支与远程分支设置为不同名字，你可以轻松地增加一个不同名字的本地分支的上一个命令

git branch -vv

git push origin  --delete serverfix   删除远程分支

git  checkout  test
git rebase master   将test中的修改同步到master中 变基

git  rebase --onto master server  client   “取出 client 分支，找出处于 client 分支和 server 分支的共同祖先之后的修改，然后把它们在 master 分支上重放一遍”。 这理解起来有一点复杂，不过效果非常酷。
git checkout master
git merge client

git rebase master server
git checkout master
git merge client