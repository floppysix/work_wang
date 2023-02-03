- 账户名：floppysix；密码 ：zilvshaonian1997
- 提交用户名以及邮箱
	- name = wangdongping
	- email = wdp319418@163.com
- **git常用指令**
|操作|指令|
|:----:|:---:| 
|初始化仓库|git init|
|查看本地库状态|git status|
|暂存本地仓库|git add hello.txt|
|提交本地仓库|git commit -m "hot-fix commit" hello.txt|
|查看版本信息|git reflog|
|查看版本详细信息|git log|
|版本穿梭|git reset --hard 版本号|
|查看分支|git branch -v|
|创建分支|git branch 分支名|
|切换分支|git checkout 分支名|
|合并分支|git merge hot-fix|
|查看当前所有远程地址别名|git remote -v|
|创建远程仓库别名|git remote add 别名 远程地址|
|推送本地分支到远程仓库|git push 别名 分支|
|克隆远程仓库到本地|git clone 远程地址|


- 当合并分支产生冲突时
	- 冲突产生的表现：后面状态为 MERGING
		- Layne@LAPTOP-Layne MINGW64 /d/Git-Space/SH0720 (**master|MERGING**)
	- 冲突产生的原因
		- 合并分支时，两个分支在同一个文件的同一个位置有两套完全不同的修改。Git 无法替我们决定使用哪一个。必须人为决定新代码内容
	- 解决冲突
		- 编辑有冲突的文件，删除特殊符号，决定要使用的内容
	- 添加到暂存区
		- git add hello.txt
	- 执行提交（注意：此时使用 git commit 命令时不能带文件名）
		- git commit -m "merge hot-fix"
	- 发现后面 MERGING 消失，变为正常
		- Layne@LAPTOP-Layne MINGW64 /d/Git-Space/SH0720 (**master**)


https://juejin.cn/post/7158258577113612302
ssh-keygen -t rsa -C "wdp319418@163.com"
Enter passphrase for key '/c/Users/wang/.ssh/id_rsa':972580


http://www.yaotu.net/biancheng/283.html
git pull --rebase guoyao master

## 上传文件
```shell
# 添加至缓存区
git add .
# 提交
git commit -m "springboot——项目打包"
# 上传
git push zongjie main

```


## 拉取文件
```shell
git pull origin main
```


## # git pull遇到错误：error: Your local changes to the following files would be overwritten by merge:
1. 如果你想保留刚才本地修改的代码，并把git服务器上的代码pull到本地（本地刚才修改的代码将会被暂时封存起来）
```SHELL
git stash
git pull origin main
git stash pop
```
2. 如果你想完全地覆盖本地的代码，只保留服务器端代码，则直接回退到上一个版本，再进行pull
```SHELL
git reset --hard
git pull origin main
```