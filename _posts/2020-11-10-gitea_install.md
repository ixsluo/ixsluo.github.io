---

title: Gitea Installation Guildline
layout: post
date: 2020-11-10 +0800

---

<!--
# Gitea Installation Guildline
-->

![shield](https://img.shields.io/badge/Git-Gitea-brightgreen?style=plastic&logo=github)

- [1. Install with Docker-Compose](#1-install-with-docker-compose)
- [2. Git使用简介](#2-git使用简介)
  - [2.1. Git工作目录](#21-git工作目录)
  - [2.2. **Git分支管理**](#22-git分支管理)
    - [2.2.1. 分支合并](#221-分支合并)
  - [2.3. **Git commit 规范**](#23-git-commit-规范)

## 1. Install with Docker-Compose

Relay on `docker`, `docker-compose`

Add user "git" to group "docker" to run docker-compose in user "git"
```sh
gpassed -a git docker
```

Docker-compose
```sh
su git
mkdir gitea ; cd gitea
cat > docker-compose.yml << EOF
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1
    container_name: gitea
    environment:
      - USER_UID=1002  # UID of git
      - USER_GID=1002  # GID of git
    restart: always
    networks:
      - gitea
    volumes:
      - /home/git/data:/data  # mount, HOST:docker
      - /etc/timezone:/etc/timezone:ro  # ro: read only
      - /etc/localtime:/etc/localtime:ro
    ports:  # port mapping, HOST:docker
      - "10080:3000"  # web for gitea
      - "10022:22"  # ssh for gitea
EOF
docker-compose up -d  # run in background
```

Visit `ip:10080/install`

(You may make a port mapping on router \<outer ip\>, 10022->10022, 10080->10080)
- simplily choose SQLite3 for database
- fill SSH domain and port with **External Domain and Port**
  - `<outer ip>:10022`
- fill HTTP port with **Internal Port**
  - default `3000`
- fill basic URL with **External URL**
  - `http://<outer ip>:10080`

Done!

## 2. Git使用简介

参考[菜鸟教程](https://www.runoob.com/git/git-tutorial.html)

```bash
# 客户端配置默认用户及邮箱、编辑器，global为全局配置
git config --global user.name "<name>"
git config --global user.email "<email>"
git config --global core.editor vim
```

### 2.1. Git工作目录

在项目文件夹内执行`git init`即将该文件夹初始化为一个git仓库。

用户在目录中所看到的即是工作区

	      add                    commit
	工  ----------------->  暂  ----------->  仓
	作  checkout -- <file>  存   reset HEAD   库
	区  <-----------------  区  <-----------  区
	^                       ^                 |
	|                       |
	|-----------------------------------------|
	          checkout HEAD <file>

|    |    |
| :- | :- |
|`git status`|                 查看当前工作区状态|
|`git add <files>`|            用工作区内的文件更新暂存区|
|`git commit`|                 将暂存区的内容更新至仓库区，**产生一个新的快照**|
|`git reset HEAD`|             将暂存区重写为仓库区保存的文件|
|`git checkout -- <file>`|     用暂存区文件覆盖工作区文件|
|`git checkout HEAD <file>`|   用仓库区文件替换暂存区与工作区|
| | |

### 2.2. **Git分支管理**

分支举例

	分支        分支名             快照树
	主分支      master    M1<----------------------M2
	预发布分支  release     \-R1<----------------R2^
	开发分支    dev            \-D1<---D2<-----D3^
	功能分支    feat               \-F1<-----F2^

快照举例

	* <hash*> (HEAD -> branch2, branch3) <commit message>
	* <hash*> <commit message>
	* <hash*> (branch1) <commit message>
	* <hash*> (master) <commit message>

*HEAD*指针表示当前所在分支及位置

|    |    |
| :- | :- |
|`git init`| 默认创建主分支*master*|
|`git branch release`| 创建*release*分支|
|`git checkout release`| 切换*HEAD*指针到*release*分支|
|`git checkout -b dev`| 创建并切换到*dev*分支|
|`git branch -d dev`| 删除*dev*分支|
| | |

#### 2.2.1. 分支合并

`git merge <待合并分支>`将待合并分支合并到当前指针所在分支。

- 快进合并，无冲突

```txt
  HEAD                            HEAD
branch1                         branch1
   |               ==>             |
   B0<--B1<--B2          B0<--B1<--B2
             |                     |
           branch2              branch
```

branch2为从branch1拉取的分支，进行两次提交到B2，**branch1没有任何提交**。

`git checkout branch1; git merge branch2`，不会产生任何冲突，直接移动*HEAD*指针和branch1到B2，快进合并，合并成功。

- 非快进合并

```txt
  HEAD                             HEAD
branch1                          branch1
   |                                |
   B0<----B1   ==>   B0<-----B1<----B2
    \-C1<-C2           \-C1<-C2<---/
          |                  |
       branch2            branch2
```

当前希望将C2合并至B1，`git checkout branch1; git merge branch2`

合并原理：追踪两个合并节点(B1, C2)最近的共同出发点(B0)，对比三方文件

1. 在B0、B1中相同，在C2中不同，说明C2是修改后需要更新的，非快进合并，合并成功，产生新节点B2
2. 存在在B0、B1、C2中都不同的文件，git不知道如何更新文件，**产生冲突**，合并失败，需要解决冲突后再合并

冲突解决：`git status`查看待解决冲突的文件名，git自动将冲突内容按下述方式标出

冲突片段举例

```txt
<<<<<<< HEAD
HEAD中的当前内容
=======
待合并的冲突内容
>>>>>>> branch2
```

修改冲突片段为需要保留的内容，`add`、`commit`后产生新的快照，再尝试`merge`合并。

### 2.3. **Git commit 规范**

`git commit [-m <message>]`进行一次提交，产生一个新的快照。每次提交必须写入提交说明，可以直接使用`-m`参数写入一行的短说明，也可以不使用`-m`，将打开一个新窗口用于编辑提交说明。

**建议使用：**`git commit -s`自动增加签名。

**建议写法：**

```txt
<type>(<scope>): <subject>
***空一行***
<body>
***空一行***
<footer>
```

1. **Header**

*type*：用于说明commit的类别，只允许以下类别

| | |
|:-|:-|
|feat    |新功能 feature |
|fix     |修复bug|
|docs    |修改文档 documentation |
|style   |不影响代码含义的改动，如代码格式等 |
|refactor|代码重构，既不新增功能，也不修复bug |
|test    |新增缺失的测试或修正已有的测试 |
|build*  |影响链接或外部依赖的改动 |
|chore   |不改变源码或测试文件的构建过程或辅助工具的改动 |
|revert* |快照回滚 |
| | |

*scope*：选填，用于说明commit的影响范围，依项目而定，可以是功能、模块、文件名等

*subject*：短描述

- 以动词开头，使用第一人称现在时
- 第一个字母小写
- 结尾不加句号（.）

2. **Body**

对本次提交的详细描述，可以是多行

3. **footer**：只用于

- 不兼容改动，以BREAKING CHANGE开头

```text
	BREAKING CHANGE: <describtion>
		<describtion>
		Before:
			...
		After:
			...
```

- 关闭issue

```text
	Closes #123
```

4. **Revert**

特殊提交，*Header*应写为`revert: <被撤销提交的Header>`