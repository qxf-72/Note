# 1. 使用 Git

## 1.1  创建仓库

 `git init` ：在当前目录建立一个 Git 仓库。

---

<br/>


<br/>


<br/>




## 1.2  添加文件

**暂存文件**：文件放入仓库所在目录中，然后 `git add filename`。

**提交文件**：使用 `git commit -m "说明文字"` 将文件添加入仓库中。可以添加 `add` 多个文件，然后一次性提交 `commit`。如下：

```bash
git add file1
git add file2 file3
git commit
```


---

<br/>


<br/>


<br/>


<br/>




# 2.  版本管理

## 2.1  查看状态

`git status` ：查看仓库当前状态。

`git diff` ：查看文件的变化。

---


<br/>


<br/>


<br/>


## 2.2 版本回退

`git log` ：查看文件的修改过程，如下：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684804260104-a75f4534-db4d-4163-a410-9cacbf8a7ba9.jpeg" width="600" /> </div>

- **版本号**：用十六进制表示，`HEAD` 表示当前版本号，`HEAD^` 表示上一个版本号，以此类推 `HEAD^^` 表示上上个版本号，`HEAD~10` 表示往上第10个版本号。
- 版本回退需要指明回退到的版本号。使用`git reset`命令，如`git reset --hard HEAD^`，回退到上一个版本。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684804817133-dc5da6a0-cc66-4f5d-aff6-73dd9c56b8b1.jpeg" width="600" /> </div>


- 在版本回退之后，需要回到当前版本更之后一点的版本，此时不能使用`HEAD`了，可以翻找之前的`log`，查到对应版本号，然后`reset`，版本号不必写全，Git会自动寻找。

- 如果之前没有使用`log`或者之前的记录丢失，无法翻找记录得到版本号时，此时使用**`git reflog`命令，显示每一次命令**。从中可以找到之前的版本号。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684805226268-8b77d988-18b9-4be0-963c-b5b2c44ae96e.jpeg" width="600" /> </div>

---


<br/>


<br/>


<br/>


<br/>



## 3.  工作区和暂存区
### 3.1  工作区

电脑中能看到的目录就工作区。

### 3.2  版本库

工作区中有一个隐藏的目录`.git`，称为版本库。其中有`stage`暂存区和`master`，以及指向`master`的指针`HEAD`。
使用`add`把文件从工作区添加到暂存区，使用`commit`把文件从暂存区提交到`master`对应分支上面。
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684805720462-920a249b-9f47-4775-a18b-319ebd0272de.jpeg" width="500" /> </div>


## 管理修改
Git跟踪并管理的是修改，而非文件本身。
对文件修改之后，如果没有`add`到暂存区，即使`commit`不会添加到`master`上面。
## 撤销修改
如果在工作区对文件进行了修改，可以使用`git checkout -- filename`来撤销对于文件`filename`的修改，此时有两种情况：

- 如果修改后还没有`add`到暂存区，工作区文件就回到和`master`一样的状态。
- 如果`add`之后做的修改，就回到和暂存库一样的状态。

总之，就是回到最近一次`add`或者`commit`的状态。

如果文件修改之后`add`了，但是还没有`commit`，使用`git reset HEAD filename`进行撤销，将文件重现放回工作区，所以`reset`即可以是版本回退，也可以是暂存区放回工作区。
## 删除文件
当在工作区删除一个文件是`rm test.txt`，`git status`会进行提示：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684810134399-32de527c-9f12-485f-862d-40f73f5a7896.jpeg" width="600" /> </div>


此时有两个选择：

- 删除版本库中的对应文件：`git rm test.txt`，然后`git commit`。
- 用版本库中的文件恢复工作区中的文件：`git checkout -- test.txt`。

---

<br/>


<br/>


<br/>


<br/>



# 远程仓库
Git 是分布式版本控制系统，同一个 Git 仓库，可以分布到不同的机器上。每台机器的版本库其实都是一样的，并没有主次之分。

实际情况往往是这样，找一台电脑充当服务器的角色，服务器仓库克隆一份到自己的电脑上，并且各自把各自的提交推送到服务器仓库里，也从服务器仓库中拉取别人的提交。Github 可以是提供 Git 仓库托管服务。

## 添加远程库

### 建立 SSH 连接

这里选择使用 SSH 将本地仓库和远程仓库连接。首先本机创建 ssh key。用户主目录之下. ssh 目录 有两个文件夹：id_rsa（私钥） 和 id_rsa. pub（公钥）。如果没有这两个目录就需要创建：
```bash
ssh-keygen -t rsa -C "邮箱地址"
```

之后登录 GitHub 添加 SSH key，把本机公钥放上去。

---

<br/>


<br/>


### 本地仓库与远程仓库关联

```bash
git remote add origin git@github.com:用户名/仓库名.git
```
- 远程库名字是 origin，可以改成别的。

本题仓库和 Github 绑定之后，使用 `git push origin main` 命令，将本机的 `main` 最新分枝推送到 Github 的对应库上面。

---

<br/>


<br/>


<br/>



## 从远程库克隆
使用命令 `git clone git@github.com:github_username/gitskills.git` 将 Github 上面的库克隆到本地。如果有多个人协作开发，那么每个人各自从远程克隆一份就可以了。

---

<br/>


<br/>


<br/>


<br/>



# 分支管理
在多人协作的工作中，可以创建一个独立分支，个人在独立分支上面创作，最后再合并分支。
## 创建与合并分支
一开始只有`master`分支，即主分支，`HEAD`并不是一定是指向主分支，而不是指向当前分支。下图展示了，创建分支并在新分支上面更改文件：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684821326786-f9d36958-bb54-4c1b-bc59-f608a029c06c.jpeg" width="600" /> </div>


如果新分支`dev`上面的工作完成了，直接将`master`指向`dev`当前提交就完成了合并。

创建分支，使用命令`git checkout -b dev`，参数`-b`表示创建并切换，相等与以下两条命令：
- `git branch dev`
- `git checkout dev`

使用`git branch`查看当前分支，当前分支之前会有一个`*`，如下图：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684821648042-7e203841-0334-4554-8153-ddaf38d74c54.jpeg" width="600" /> </div>



当需要合并分支时，先切换回到主分支上面，然后使用`git merge dev`将分支`dev`合并到当前分支上面。合并分支之后，就可以删除原来的分支了`git branch -d dev`。
## 解决冲突
当创建分支之后，不同分支对于同一个文件都做出来修改，此时进行合并分支操作，Git无法执行“快速合并”，只能试图把各自的修改合并起来，但这种合并就可能会有冲突。如下图：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684841276657-5255df96-cbdb-4fc8-b46c-c99630ec4aab.jpeg" width="600" /> </div>


此时打开冲突的文件，可以发现，`====`和`<<<<<`分隔开了不同分支提交的内容。
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684841331107-6773fe71-6fb2-4cd6-a428-9c8e0d3f2b00.jpeg" width="600" /> </div>



此时需要处理冲突，在主分支上面修改文件然后`add`和`commit`。此时并没有完全合并成功（分支指针并没有指向相同地方），但是在主分支上面已经有合并了`dev`分支的内容，所以可以将`dev`分支删除。

可以使用`git log --graph`查看分支合并图：
<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2023/jpeg/29674612/1684841818626-29c731d9-8027-4321-a541-8e9a37e1a0a9.jpeg" width="600" /> </div>


---











