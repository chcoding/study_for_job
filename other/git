## Git

### git参考文档

查看具体命令的帮助文档

git help <具体命令>

eg: git help branch



mac下oh_my_zsh下缩写git命令缩写参考

https://blog.csdn.net/weixin_34315189/article/details/88971140



### 本地项目上传github

> 1.首先在项目目录下初始化本地仓库
> git init
>
> 2.添加所有文件( . 表示所有)
> git add .
>
> 3.提交所有文件到本地仓库
> git commit -m "备注信息"
>
> 4.连接到远程仓库
> git remote add origin 你的远程仓库地址
>
> 5.将项目推送到远程仓库
> git push -u origin master

如果出现无法连接的问题：Please make sure you have the correct access rights and the repository exists

表示电脑和github的ssh连接有问题，需要在电脑产生ssh密钥，并且把公钥添加到github中。

```shell
//-C 注释 -t 指定密钥类型
ssh-keygen -C 'comment' -t -rsa
```

https://blog.csdn.net/jingtingfengguo/article/details/51892864



### github项目工作流程

1. `git clone` // 到本地
2. `git checkout -b feature ` 切换至个人feature分支（相当于复制了remote的仓库到本地的xxx分支上
3. 修改或者添加本地代码（部署在硬盘的源文件上）
4. `git diff` 查看自己对代码做出的改变
5. `git add` 上传更新后的代码至暂存区
6. `git commit` 可以将暂存区里更新后的代码更新到本地git
7. `git push origin feature` 将本地的feature分支上传至github



**feature分支开发过程中远端master分支有新的提交**

1. `git checkout master` ：切换回master分支
2. `git pull origin master` ：将远端修改过的代码再更新到本地
3. `git checkout feature` ：回到feature`分支
4. `git rebase main`： 等效于在master最新提交上追加feature的修改
5. `git push -f origin xxx`： 把rebase后并且更新过的代码再push到远端github上
   （远程feature中没有master分支最新的提交，有冲突，需要用-f强制提交）
6. 提交`pull request` ，采用 `squash and merge`将feature分支所有的commit合并为一个（**简洁**）。



**远端完成更新后**

1. `git branch -D xxx`： 删除本地的git分支
2. `git pull origin master` ：把远端master分支的最新代码拉至本地



### Git常见命令用法

##### git rebase用法

`rebase` 通常用于重写提交历史。下面的使用场景在大多数 Git 工作流中是十分常见的：

- 我们从 `master` 分支拉取了一条 `feature` 分支在本地进行功能开发
- 远程的 `master` 分支在之后又合并了一些新的提交
- 我们想在 `feature` 分支集成 `master` 的最新更改

https://waynerv.com/posts/git-rebase-intro/

常规用法：

> git rebase master //合并master分支最新的提交合入本分支，本分支就像在最新的master分支进行特征开发
>
> git rebase -i  commit_id  //变基，对历史交替做修改



##### 修改最近的一次提交信息

当仅仅只想修改最近的一次提交时，使用 `git commit --amend` 会更加方便。

它适用于以下场景：

- 我们刚刚完成了一次提交，但还没有推送到公共的分支。
- 突然发现上个提交还留了些小尾巴没有完成，比如一行忘记删除的注释或者一个很小的笔误，我们可以很快速的完成修改，但又不想再新增一个单独的提交。
- 或者我们只是觉得上一次提交的提交信息写的不够好，想做一些修改。

这时候我们可以添加新增的修改（或跳过），使用 `git commit --amend` 命令执行提交，执行后会进入一个新的编辑器窗口，可以对上一次提交的提交信息进行修改，保存后就会将所做的这些更改应用到上一次提交。



##### Git撤销操作

取消本地的修改：`git checkout <file>` |`git restore <file>`

只取消暂存区的修改：`git reset <file>`|`git restore --staged <file>`

取消本地和暂存区的修改：`git checkout HEAD <file>`

取消已commit的修改：

- reset

`HEAD~1` ：上个版本

1. `git reset --soft HEAD~1`：只取消commit的内容，stage区和本地修改都不丢失
2. `git reset --mixed HEAD~1`：取消commit的内容、stage区的修改，本地修改都不丢失
3. `git reset --hard HEAD~1：`所有修改丢失，直接回退到上个版本



- revert

`git revert HEAD` ：重新提交一个commit，并取消HEAD版本中的修改，类似提交与HEAD相反的操作。

![1667651793361](C:\Users\Chenhui\AppData\Roaming\Typora\typora-user-images\1667651793361.png)



两者区别

`git reset`是取消修改版本的修改，而`git revert`是额外提交一个撤销的commit。

如果修改已经提交到了master，在个人分支使用`git revert`（额外新增一个撤销效果的提交），而不是`git reset`，否则master合并个人分支有冲突。