# wsl使用笔记

## 镜像网络

wsl默认使用桥接（bridge）模式，即虚拟机和宿主机之间通过网桥进行通信。但是这样不方便我们使用代理，这时可以给wsl设置镜像模式，，此时需要打开`C:\\Users\\你的用户名\\.wslconfig`并写入以下内容

```plaintext
[wsl2]
autoProxy=true
networkingMode=mirrored
```

然后重启wsl，即可应用wsl模式。


## wsl的导出

wsl默认在C盘，有时候C盘空间不够，系统会直接把wsl的文件映射路径删除了（我被坑了好几次了），因此需要把wsl导出到别的盘，比如D盘。

具体命令如下，把发行版名称和路径替换为你自己的即可：\
`wsl --export Ubuntu-22.04 D:\WSL\ubuntu2204_backup.tar`

确认导出后，需要注销原发行版。
`wsl --unregister Ubuntu-22.04`

最后使用导出命令导出，将路径名替换为自己的即可：
`wsl --import Ubuntu-22.04 D:\WSL\ubuntu2204 D:\WSL\ubuntu2204_backup.tar --version 2`


## 设置默认用户

linux的root用户默认都是没有各种高级终端功能的，这导致你装了zsh也是白装。而wsl2导出到别的盘后（节省C盘空间），默认都是以root用户进入的，这就很大限制了我们在命令行操作下的效率。因此我们需要给wsl2设置默认用户。

进入wsl后，使用`vim /etc/wsl.conf`并添加:

```plaintext
[user]
default=username  # 替换为你的非root用户名

```
重启后即可应用。

## win下操作wsl用户

在设置默认用户后，我很尴尬地忘掉了root用户的密码，不过好在wsl允许在windows下直接登录该用户，于是我们可以修改用户密码。

可以使用
```wsl -u root -d Ubuntu-22.04  // 你的发行版```

以root 进入你的wsl中，然后键入：

```cat /etc/passwd```

这时会输出一个用户列表，其中包含了用户名、密码、用户ID、组ID等信息，其中每行代表一个用户，格式为：用户名:密码占位符:用户ID:组ID:注释:家目录:默认shell

然后使用`passwd your_user `，根据提示重置密码




## 调用Win浏览器
笔者在使用wsl开发时，遇到了需要使用`xdg-open`打开浏览器的情况，然后喜提```/usr/bin/xdg-open: 882: firefox: not found```等一堆报错。因为wsl没有安装浏览器，所以xdg找不到浏览器，这时候有两个操作：

1. 安装火狐浏览器或其他浏览器
2. 改变xdg的配置，使其打开Win的浏览器，具体方法如下

在wsl中，win的默认Edge浏览器路径为`/mnt/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe`可以使用

`ls "/mnt/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe"`   判断是否存在，然后输入：

```bash
sudo vim /usr/local/bin/win-browser
```

在其中写入

```bash
#!/bin/bash
"/mnt/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" "$@" &
```

然后是常规操作

```bash
sudo chmod +x /usr/local/bin/win-browser
echo 'export BROWSER=win-browser' >> ~/.bashrc
source ~/.bashrc  # 立即生效
```

测试： `xdg-open http://localhost` ，如能打开浏览器，说明调用成功
