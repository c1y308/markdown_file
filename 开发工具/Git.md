# 使用方法

把 **Git 的提交历史** 想象成一条**时间线**：

- 每一次 `git commit` = 时间线上的**一个节点**（有唯一编号：哈希值）
- **分支 (branch)** = 贴在节点上的**便利贴**（写着 `main`/`dev`/`feature`，轻到几乎不占空间）
- **HEAD 指针** = 你的**手指**（指向你当前正在编辑的位置），是 Git **唯一**的「当前位置指针」，一般让 HEAD 指针跟着分支移动。
- **GitHub 远程仓库** = 云端的**另一条一模一样的时间线**（有自己的便利贴）

---

**分支：**只是一个指针，**分支就是一个指向 commit 节点的指针**，和便利贴一样，想贴哪个节点就贴哪个。

举例：
	1.第一次提交：commit A → 自动创建默认分支 main，    main 便利贴 → 贴在 commit A 上。
	2.第二次提交：commit B → main 便利贴自动向后移动，main 便利贴 → 贴在 commit B 上。

---

## 头指针分离

一般让 HEAD 指针跟着分支移动，而 HEAD 指针直接指向了一个 commit 节点，而不是指向任何分支，就会发生头指针分离。

一般以下情况会发生头指针分离：

``` shell
# 直接切换到某个历史提交
git checkout commit哈希值

# 直接切换到标签 (tag)
git checkout v1.0
```

在这个状态下，**可以修改代码、提交新 commit**，但：这些新提交是**无主的**（没有任何分支指向它）！

一旦切回正常分支（比如 `git checkout main`）：HEAD 重新指向分支 → 这些无主提交会被 Git 当成「垃圾」回收 → **代码直接丢失！**

想保存分离头的修改：**立刻新建分支固定提交**：

``` shell
git checkout -b 新分支名
```

---

如果使用`git push origin HEAD:<远程分支名>`强行提交会导致灾难：把`master`指针指向commit B：

``` shell
新的远程 master 指针
    ↓
A → B  C → D  (C、D 还在！但被 master 抛弃了)
```

✅ **commit C、D 没有被删除**（还存在于 Git 仓库里）

❌ **远程 master 分支的历史里，再也看不到 C、D 了**：Git 的提交历史，是【单向向后追溯】的，不是正向看的！每个 commit **只记录自己的「父节点」（上一个提交），从不记录「子节点」（下一个提交）**分支指针指向谁，就只从这个 commit 往回翻历史，永远不会往下找后续的 commit。

别人拉取 master 代码，只能拿到 A→B，C、D 直接消失！

补救方法：

``` shell
# 1. 给分离头的新提交新建临时分支，「固定」提交（防止丢失）
git checkout -b temp_motor_id

# 2. 切回本地 main 分支（如果本地 main 不存在，先创建并关联远程：git checkout -b main origin/main）
git checkout main

# 3. 把临时分支的修改合并到 main（快速合并，无冲突）
git merge temp_motor_id

# 4. 正常推送到远程 main（标准推送，不会覆盖远程历史）
git push origin main

# 4.1 如果有冲突，解决之后：
git commit -m "merge temp"


# 5. （可选）删除临时分支
git branch -d temp_motor_id
```

退出分离头指针：

``` shell
git checkout main
```

## 合并分支

`git merge` 的本质：把「另一个分支的所有提交」，合并到「当前所在的分支」，**最终只移动当前分支的指针**。

- 它**不删除、不修改**任何已有的 commit

- 它只做两件事：**拼接提交历史** + **移动分支指针**

- 合并前必须先切换到**接收合并的分支**（比如要把 dev 合进 main，就先切到 main）

``` shell
git checkout main
git merge dev
```

---

### 单线合并

两个分支**没有产生分叉**，只是一条直线：

1. `main` 分支：`A → B`
2. 从 `B` 新建 `dev` 分支，在 `dev` 上开发：`A → B → C → D`
3. 期间 `main` 分支**完全没有任何修改**

Git 检测到`main` 的最新提交 `B`，是 `dev` 分支的**祖先节点**（dev 包含了 main 的所有代码）

→ **直接把 main 指针「快进」到 dev 的最新提交 `D`**

→ **不创建任何新的 commit！**

``` shell
# 合并前
    dev
    ↓
A → B → C → D
    ↑
    main

# 合并后（快进）
            main, dev
            ↓
A → B → C → D
```

### 三方合并

``` shell
     D → E  (你的分支：feature)
    /
A → B → C  (最新主线：main，别人更新了)
```

把**最新的主线代码（C）合并到你的分支里**，让你的分支包含最新主线，再合回主线，相当于：**先同步主线到你的分支，解决冲突，再推给团队**。

``` shell
# 1. 切换到主线分支
git checkout main

# 2. 拉取远程最新主线代码（把本地main更新到最新的C节点）
git pull origin main

# 3. 切回你的开发分支
git checkout feature

# 4. 【关键】把最新的主线代码，合并到你的分支里
git merge main
```

合并后的分支：

``` shell
              F (合并提交)
             /\
        D → E  \
       /        \
A → B → C →──----\
```

最后合入主线并推送：

```shell
# 切回主线
git checkout main

# 合并你的分支（此时已经兼容最新主线，无任何风险）
git merge feature

# 推送到远程GitHub，完成！
git push origin main
```

## 合并仓库

### 无本地仓库

你的**电机主仓库**：`~/Myactua_Ethercat`

你有 **IMU 远程仓库地址**（去你的代码托管平台：GitHub/Gitee/GitLab 复制克隆地址）

目标：把远程 IMU 代码 → 电机仓库的 `/src/imu/a100` 文件夹

---

直接关联远程 IMU 仓库，合并代码，一步到位：

``` shell
# 1. 进入电机主仓库
cd ~/Myactua_Ethercat

# 2. 添加远程IMU仓库（取名 imu）
git remote add imu 你的IMU远程仓库地址

# 3. 拉取远程IMU的所有代码和提交历史
git fetch imu

# 4. ✅ 核心：将IMU代码(远程仓库) 直接合并(绑定)到 src/imu/a100 目录
# 自动创建所有文件夹，无需手动新建！
# 如果IMU分支是 master，把 main 改成 master
git subtree add --prefix=src/imu/a100 imu main --squash

# 5. 推送到远程电机仓库（保存合并结果）
git push origin main
```

`fatal: working tree has modifications. Cannot add.` 的核心是：`git subtree add` 命令**强制要求工作区完全干净（无任何未提交的修改、无未跟踪文件）**。要么提交更改，要么放弃更改，要么使用`git stash`暂存所有未提交的修改。

---

想删除 IMU 远程关联（后续不用了）：

``` shell
git remote remove imu
```

**只更新imu代码：**

✅ 不是普通的 `git pull imu`

✅ 而是 **`git subtree pull`**（subtree 专属更新命令）:

````shell
git subtree pull --prefix=src/imu/a100_imu imu master --squash
````

- `--prefix=src/imu/a100_imu`：告诉 Git → **只更新这个绑定的文件夹**
- `imu`：你关联的 IMU 远程仓库名
- `master`：IMU 仓库的分支
- `--squash`：把更新压缩成 1 条提交，保持日志干净

执行后，会**自动把 IMU 远程仓库的最新代码，拉取并覆盖到 `src/imu/a100_imu` 文件夹里**。

**推送imu代码：**

``` shell
# 反向推送：只把电机仓库里的imu代码 → 推回 IMU 远程仓库
git subtree push --prefix=src/imu/a100_imu imu master
```

#  安装与初始化

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

#  本地仓库

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

查看版本号与回滚：

``` bash
git log test.c
git reset 1a1e91bf37add6c3914ebf20428efc0a7cea33f3 test.c  // 只回滚暂存区的文件
git reset --hard 1a1e91bf37add6c3914ebf20428efc0a7cea33f3 test.c // 同时回滚暂存区与工作区的文件

git checkout test.c // 直接将工作区的文件回滚到上一版本
```

# 远程仓库

`origin`：

- **默认远程仓库**：当使用 `git clone` 克隆仓库时，Git 会自动将源仓库命名为 `origin`
- **有推送权限**：因为这是你的仓库，可以直接 `git push origin 分支名`

``` bash
# 链接到远程仓库（git会自动执行）
git remote add origin git@gitee.com:cjysdl1/quard-star.git

git remote set-url origin git@github.com:yourname/new-repo-name.git
# 将本地分支推送到远程的 origin 仓库。
git push -u origin 本地分支名
# 之后只需使用：
git push / git pull
```

查看远程仓库：

``` bash
git remote -v
```

# 克隆远程仓库

`clone`远程仓库**不需要在本地`init`一个仓库**，也不用`git checkout -b master`来创建分支。

`git clone` 命令会在 `home/a/` 目录下创建一个与远程仓库同名的子文件夹，Git 文件（`.git` 目录）就在这个子文件夹内。

`git branch -r`查看远程有哪些分支。

# 提交一个PR

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
