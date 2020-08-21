---
title: git常用命令总结
copyright: true
date: 2018-11-14 16:47:04
tags: git命令总结
categories: git

---

# 克隆代码

将远端的git项目拉取到本地：

````
git clone resource_url

````


## 正常的开发流程

````
1.在master分支拉取远端分支的代码到本地，以保证新创建的分支的时候是最新的代码；
git pull origin master 
 
2.查看最新的tag，因为开发分支名字的后缀tag是基于最新的tag开出来的，加入最新的tag是1.1.1
git tag

3.创建新的分支 

git checkout -b br_test_develop_1.1.1

4.进行开发
。。。。

5.查看代码的状态
 git status
 
6.'.'代表当前目录下的所有文件， 也是可以只是添加某一个文件到stage缓存区
git add . 或者 git add files 

7.把 stage 缓存生成一次 commit，并加入 commit 历史, - m是添加注释
git commit -m "commit message"

8.将新的分支push到远端，将新的commit代码push到远端。
git push origin br_test_develop_1.1.1

````


## 修改提交日志

假如你执行 commit -m 时记录的日志描述和本次执行commit的功能不符合，又不想创建新的commit，此时可以使用一下命令。

````
git commit --amend  

然后使用 vim编辑操作

````

## 上线项目需要做的操作

````
这里假设开发分支为 br_test_develop_1.1.1 。

1. 先拉取br_test_develop_1.1.1的远端代码，
git pull origin br_test_develop_1.1.1

2. 然后切换到master分支,拉取master分支的远端代码
git checkout master
git pull origin master

3. 切换到br_test_develop_1.1.1分支，merge master的代码到br_test_develop_1.1.1,有冲突解决冲突，提交到远端。（防止开发的代码污染master分支，这一步不可以省略。）
git checkout br_test_develop_1.1.1
git merge master
git push origin br_test_develop_1.1.1

4. 切换到master分支，merge br_test_develop_1.1.1的代码到master分支。
git checkout master
git merge br_test_develop_1.1.1
git push origin master

5. 确认当前分支是master，且是最新的代码。查看tag，上线的tag在最新的基础上命名一个，或者按照版本需要创建一个tag。
git fetch -all // 这将更新git remote 中所有的远程repo 所包含分支的最新commit-id，包括tag
git tag  // 查看tag 列表
git tag 1.1.2  //创建本地tag为1.1.2
git push --tag // push tag 到远端

````

## 情景1：本地开发着，需要合并远端的开发分支的代码

````
1. stash存储本地的开发中的代码
git add .
git stash

2. 执行rebase命令，拉取远端代码重建base
git pull --rebase

3. 取出stash的代码，pop命令会取出并删除stash的代码（最近的一次）
git stash pop 

4. 手动解决冲突 并add .
git add .

5. 结束reabse模式，如果此时提示No rebase in progress?则表示已经没有冲突了；否则上面两步要重复多次或者执行 git rebase --abort
git rebase --continue 或者 git rebase --abort

6. 提交代码到远端
git status
git add .
git commit -m “commit message”
git push origin 'branchName'

````
## 情景2：git reset 回退到某次提交的代码

````
1. 查看log日志，取出要取消的某次操作的commitId
git log

2. 执行reset操作 
git reset --hard <commitId> 
或者 
git reset --hard HEAD~2 //回退两个commitId


3. 强制提交到远端 (-f 慎重使用)
git push origin <branchName> -f

````

`注意`：reset参数。<b>add 是将代码添加工作区，commit是将代码添加到缓存区</b>

- -soft – 缓存区和工作目录都不会被改变(执行了add和commit)
- --mixed – 默认选项。缓存区和你指定的提交同步，但工作目录不受影响（只是add 未commit） 
- --hard – 缓存区和工作目录都同步到你指定的提交(代码都没有了)（代码都没有了）



## 情景3： git revert 取消某次的提交(merge版本的commit无法处理)

````
1. 查看log日志，取出要取消的某次操作的commitId
git log

2. 执行reset操作
git revert -n <commitId>

3. commit push
git commit -m “commit message”
git push origin <branchName>

````

## 情景4：本地正开发着，需要切换到别的分支，此时可以将代码stash到缓存区

git-stash - Stash the changes in a dirty working directory away

````
git add .

git stash //Stash the changes in a dirty working directory away

git stash list //查看缓存记录

//需要恢复缓存的代码到工作区. 其中stashId可以在git stash list 命令展示的列表中查看
git stash pop 或者 git stash apply stashId

// 清除缓存 
git stash clear  或者 git stash drop  stashId

````

`建议`： stash缓存使用完请及时clear，时间久了可能容易忘记stash的内容，恢复的时候造成冲突。


## 关于分支的常用操作

````
1. 拉取远端的分支代码，因为默认down代码的时候，不是获取所有远端的最新的代码
 git fetch --all

2. 查看远端、本地的分支，以及所有分支
git branch //查看本地分支
git branch -r // -r代码remote，所有的远端分支。
git branch -a //查看所有的分支

3. 一般情况下，远端分支和本地分支是同名的，如果不是同名的需要查看本地分支追踪的远端分支如何查看。
git branch -vv

4. 如何设置本地追踪的远端分支
git branch --set-upstream-to=origin/<branch> master

5. 删除在本地有但在远程库中已经不存在的分支
git remtte prune origin 

6. 删除本地分支
git branch -d <branchName>

7. 删除远端分支 (必须有权限)
git push origin --delete <branchName>  


````

## 关于标签的操作

````
1. 查看标签列表 之前要执行git fetch --all 拉取远端的分支以及tag到本地
git tag

2. 查看tag的详细信息,打标签的人以及时间
git show <tagName>

3. 打标签
git tag <tagName> // 在本地的当前commitId节点处创建一个tag标签
或者
git tag -a <tagName> -m "commit message"

4. 删除本地标签
git tag -d <tagName>

5. 删除远端分支
git push origin --delete <tagName>

````


## 创建新的项目提交到gitlab上

1. 需要在gitlab上操作创建一个项目
2. 配置用户名和邮箱

### 已有项目添加到远端

````
1. 计入文件夹，初始化为git项目
cd existing_folder
git init

2. 本地和远端进行关联
git remote add origin http://172.16.117.224/ent_teach/ent-test.git

3. 执行 add commit push 操作
git add .
git commit -m "Initial commit"
git push -u origin master

````
### 创建一个新项目

````
1. 克隆远端项目
git clone http://172.16.117.224/ent_teach/ent-test.git

2. 进入相应的目录，添加README.md文件。执行 add commit push 操作
cd ent-test
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

````

## 解决冲突的技巧

````

这一篇解释了手动解决冲突时<<<<和=====的含义 
一般情况下rebase都是会有冲突的，详细查看冲突可以用命令git status然后就会显示哪个文件有冲突，然后打开有冲突的哪个文件，会发现有一些“<<<<<<<”， “=======”， “>>>>>>>” 这样的符号。

“<<<<<<<” 表示冲突代码开始

“=======” 表示test与master冲突代码分隔符

“>>>>>>>" 表示冲突代码的结束

<<<<<<<  
所以这一块区域test的代码

=======  
这一块区域master的代码

>>>>>>> 

````

## 其它

````
1. 查看日志
git log  //显示的是commitId以及commit Message

2.查看状态
git status

3. 快速查看旧版本
git checkout commitId

````

## gitlab配置ssh key

1.打开本地git bash,使用如下命令生成ssh公钥和私钥对

````
ssh-keygen -t rsa -C 'xxx@xxx.com' 然后一路回车(-C 参数是你的邮箱地址)
````
![ssh-key](/images/git/ssh-key.png)

2.然后打开~/.ssh/id_rsa.pub文件(~表示用户目录，比如我的windows就是C:\Users\Administrator)，复制其中的内容

3.打开gitlab,找到Profile Settings-->SSH Keys--->Add SSH Key,并把上一步中复制的内容粘贴到Key所对应的文本框，在Title对应的文本框中给这个sshkey设置一个名字，点击Add key按钮

![gitlab-setting](/images/git/gitlab-setting.png)

4. 到此就完成了gitlab配置ssh key的所有步骤，我们就可以愉快的使用ssh协议进行代码的拉取以及提交等操作了


## 参考
1. [git fetch 与git pull的区别](https://www.cnblogs.com/ToDoToTry/p/4095626.html)
2. [git rebase的用法](https://blog.csdn.net/TTKatrina/article/details/79288238)
3. [git revert与git reset操作详解](https://blog.csdn.net/yxlshk/article/details/79944535)
4. [暂存区与工作区](https://www.cnblogs.com/ouber23/articles/5466040.html)
