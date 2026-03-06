#  1.安装与初始化

``` bash
sudo apt install git

sudo apt search git
```

设置当前机器Git的标识：

``` bash
git config --global user.name "cjysdl"
git config --global user.email "755381914@qq.com"
```

获取ssh公钥：

``` bash
ssh-keygen -t ed25519 -C 755381914@qq.com

ls ~/.ssh/

cat ~/.ssh/id_ed25519.pub
```

#  2.本地仓库

创建本地仓库：

``` bash
git init
git checkout -b master  // 创建分支

git branch  // 查看分支
```

将文件提交/删除到缓存区：

``` bash
git add test.c //将文件先添加到提交缓存区
git add --all  // 将所有工作区改动给的文件提交到缓存区

git rm test.c // 不仅可以从工作区中删除文件，还可以将文件的删除操作记录到暂存区

git reset --hard // 删除commit之前所有对缓存区的操作
```

查看缓存区与工作区的文件有无变化：

``` bash
git status
```

将缓存区的提交到本地仓库：

``` bash
git commit -m "add new file \"test.c\""  // 将提交/删除的缓存区的所有文件提交到本地仓库
git commit --amend // 修改上一次提交的备注信息
```

``` bash
git log  // 查看仓库提交历史的版本号
git reflog // 仓库的提交历史

git reset --hard 版本号
git reset --hard HEAD^  // 将HEAD指向的仓库回滚到上一版本
git reset --hard HEAD~3  // 将HEAD指向的仓库回滚到上三个版本
```

查看**中文件**版本号与回滚：

``` bash
git log test.c
git reset 1a1e91bf37add6c3914ebf20428efc0a7cea33f3 test.c  // 只回滚暂存区的文件
git reset --hard 1a1e91bf37add6c3914ebf20428efc0a7cea33f3 test.c // 同时回滚暂存区与工作区的文件

git checkout test.c // 直接将工作区的文件回滚到上一版本
```

# 3.远程仓库

`origin`：

- **默认远程仓库**：当使用 `git clone` 克隆仓库时，Git 会自动将源仓库命名为 `origin`
- **有推送权限**：因为这是你的仓库，可以直接 `git push origin 分支名`

``` bash
# 链接到远程仓库（git会自动执行）
git remote add origin git@gitee.com:cjysdl1/quard-star.git

# 将本地分支推送到远程的 origin 仓库。
git push -u origin 本地分支名
# 之后只需使用：
git push / git pull
```

查看远程仓库：

``` bash
git remote -v
```

# 4. 克隆远程仓库

`clone`远程仓库**不需要在本地`init`一个仓库**，也不用`git checkout -b master`来创建分支。

`git clone` 命令会在 `home/a/` 目录下创建一个与远程仓库同名的子文件夹，Git 文件（`.git` 目录）就在这个子文件夹内。

`git branch -r`查看远程有哪些分支。

# 5. 提交一个PR

首先 Fork 项目：

- 访问项目 GitHub 页面
- 点击右上角的 **Fork** 按钮
- 这会创建项目副本到你的 GitHub 账户

然后：

```sh
# 将fork的项目 clone 到本地
git clone https://github.com/你的用户名/项目名.git
cd 项目名

# 把原始仓库命名为 upstream 进行链接，方便追踪原仓库的更新
git remote add upstream https://github.com/原始作者/项目名.git

# 1. 确保本地/远程仓库的 main 分支是最新的"版本"
git checkout main
git fetch upstream
git merge upstream/main        # 现在 main = 官方最新代码
git push origin main           # 同步到你的 GitHub fork
# 2. 切换到正在开发的功能分支
git checkout feature-xxx
# 3. 将最新官方代码合并到你的功能分支（二选一）
# 方案 A：合并（merge）- 保留历史，有合并提交
git merge main
# 方案 B：变基（rebase）- 线性历史，无合并提交，更干净
git rebase main


# 创建新分支
git checkout -b 你的分支名

# 进行代码修改后
git add .
git commit -m "描述性提交信息"

git push origin 你的分支名
```
