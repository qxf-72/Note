
系统：Ubuntu 20.04


# 登录密码



忘记登录密码，重置密码方法：
- 关机重启
- 按 shit 键
- 选择 Advanced options for Ubuntu，进入 Recovery Mode
- 进入进入 root shell
- 重新挂载根目录为可写：mount -o remount,rw /
- 修改用户密码：passwd xiaofeng



---

<br/>


<br/>


<br/>


<br/>


# 远程连接

## 建立连接

- 使用 `ifconfig` 查看本机 ip 地址，如果没有该命令，使用 `sudo apt install net-tools` 安装工具。
- `ssh xiaofeng@192.168.213.135`，测试连接
- 为了避免每次都输 IP 地址 ：
	- 生成本地公钥 `ssh-keygen -t ed25519 -C "your_email@example.com"`，一般保存在 user/用户名/.ssh/id_rsa.pub 里面
	- 将公钥复制到远程主机，打开 ~/.ssh/authorized_keys，将公钥添加进去
	- 配置 ~/.ssh/config
```text
Host 远程主机名字  
HostName 192.168.1.101  
User xiaofeng  
Port 22  
```

- 之后 `ssh 主机名字` 就可以连接了。


## 文件传输


从本地传到远程：

```bash
scp <文件名字> xiaofeng@192.168.213.135:<目标目录>
```

从远程下载到本地：

```bash
scp xiaofeng@192.168.213.135:<文件名字> <目标目录>
```


---


<br/>


<br/>


<br/>


<br/>


# apt

apt（Advanced Packaging Tool）是一个在 Debian 和 Ubuntu 中的 Shell 前端软件包管理器。




---


<br/>


<br/>


<br/>


<br/>


# 配置 Go 环境

## 下载 VScode

推荐使用 apt 安装：

安装依赖
```bash
sudo apt update
sudo apt install wget gpg
```


导入微软密钥

```bash
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages. microsoft. gpg
sudo install -D -o root -g root -m 644 packages. microsoft. gpg /etc/apt/keyrings/packages. microsoft. gpg
```


添加仓库
```bash
echo "deb [arch=amd 64 signed-by=/etc/apt/keyrings/packages. microsoft. gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources. list. d/vscode. list > /dev/null
```

安装 VSCode
```bash
sudo apt update
sudo apt install code
```

---

<br/>


<br/>


<br/>


## 安装 go

**不推荐使用 apt 下载，其版本过于老旧**。

下载

```bash
cd /tmp
wget https://go.dev/dl/go1.26.2.linux-amd64.tar.gz
```

删除旧目录
```bash
sudo rm -rf /usr/local/go
```

解压安装
```bash
sudo tar -C /usr/local -xzf go 1.26.2. linux-amd 64. tar. gz
```

配置 PATH，打开：

```bash
nano ~/. bashrc
```

加入：

```bash
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH= $PATH:$ GOPATH/bin
```

保存后：

```bash
source ~/. bashrc
```

验证


```bash
go version
```

---

<br/>


<br/>


<br/>


## 配置 vscode 

在插件商城搜索 **go 插件** ，并安装。

---

先配置**国内 Go 代理**，在终端执行：

```bash
go env -w GOPROXY=https://goproxy.cn,directgo env -w GOSUMDB=sum.golang.google.cn
```

检查：

```bash
go env GOPROXYgo env GOSUMDB
```

应该看到：

```bash
https://goproxy.cn,directsum.golang.google.cn
```

---


Ctrl + Shift + P 打开命令面板快捷键，输入 Go: Install/Update Tools ，把所有选项全部选上，点 ok 进行**安装开发工具**。

---

在万能键输入 Preferences: Open User Settings (JSON) ，**打开设置文件**，添加：

```json
{
    "editor.formatOnSave": true,
    "[go]": {
        "editor.defaultFormatter": "golang.go"
    },
    "go.formatTool": "goimports"
}
```

---

安装**其他工具**：
- goimports：做了gofmt 的工作，还能自动添加/删除 import
- staticcheck：高级静态分析
- dlv：官方调试器

在 bash 中，输入：
```bash
go install golang.org/x/tools/cmd/goimports@latest
go install honnef.co/go/tools/cmd/staticcheck@latest
go install github.com/go-delve/delve/cmd/dlv@latest
```


----

<br/>


<br/>


<br/>


<br/>



# 配置 Cpp+Cmake 环境



---

<br/>


<br/>


<br/>


<br/>


# 变更时区


```bash
sudo timedatectl set-timezone Asia/Shanghai
```

---
