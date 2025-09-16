# linux入门：如何有效清理垃圾文件
<style>
    p {
        text-indent: 2em;
    }

    img {
        display: block;
        margin: 0 auto;
        width: 85%;
    }

</style>
由于笔者将wsl挂载到了只有150G的F盘，空间资源明显不足，亟需垃圾清理。
![](https://picx.zhimg.com/80/v2-b0ed921c25d2fef978b0f9be6bbfcc2f_1440w.png)


## 空间占用查看

清理垃圾首先需要查看各个目录的空间占用情况，可以使用`du`命令查看。这里比较推荐的是
```bash
du -hd 1 dir
```
这一命令，其中`-h`选项将文件大小以更易读的方式显示，`-d`选项指定展示深度，`1`表示显示深度，`dir`表示要查看的目录。这样可以清晰地看到各个目录的大小。
```bash
❯ sudo du -hd 1 /root
1.8G    /root/go
21G     /root/.cache
8.0K    /root/.vim
12K     /root/.codeverse
8.0K    /root/.fitten
1.9G    /root/.trae-cn-server
731M    /root/.local
5.0M    /root/.trae-aicc
272K    /root/.vscode-remote-containers
8.0K    /root/.trae-cn
9.5M    /root/.docker
4.0K    /root/.imageio
```

## 垃圾文件分类

对于linux系统，垃圾文件主要分为三类：
1. 临时文件：系统在运行过程中产生的临时文件，如`*.tmp`、`*.swp`等。
2. 缓存文件：系统运行过程中产生的缓存文件，如`*.cache`，主要在`~/.cache`下。
3. 日志文件：系统运行过程中产生的日志文件，如`*.log`、`*.err`等。

## 垃圾文件清理

### 日志文件清理

> 可参考[Arisa的博客](https://blog.arisa.moe/blog/2022/220417-linux-clean-up-disk-space/#journal)


使用以下查看并清理日志文件
```bash
journalctl -x --disk-usage
journalctl --vacuum-size=10M  # 清理日志到只剩下 10M
journalctl --vacuum-time=1d   # 清理一天前的日志
```

注意，不要用`rm`命令来删除日志文件，因为很有可能有其他进程在向其中写入文件，导致删除失败。

以上只是一次性的清理，如果需要永久限制日志文件大小，可以修改`/etc/systemd/journald.conf`文件，移除`SystemMaxUse`和`RuntimeMaxUse`的注释在后方添加如下内容

```bash
[Journal]
SystemMaxUse=10M   # 硬盘中只保留最近 10M 的日志
RuntimeMaxUse=10M  # 内存中只保留最近 10M 的日志
```

除了系统日志，还有一些其他日志文件，如`/var/crash`目录下存放系统崩溃信息，`/var/log/apt`目录下存放apt相关日志，`/var/log/alternatives`目录下存放软件链接信息等。这些文件一般占用不大，清理了也不会腾出太多空间。


### 缓存文件清理

#### 包管理器的自动清理   

对于`apt`包管理器，可以使用以下命令清理：

```bash
sudo apt clean
sudo apt automove --purge #purge选项可以同时删除配置文件
```

如果常使用`conda`,`uv`等python包管理器，还会在`~/.cache`中找到很多缓存文件，可根据`du -hd 1 ~/.cache/uv`的结果酌情清理。

### 临时文件清理

`/tmp`下的临时文件一般会自动清理，可以查看你的`/usr/lib/tmpfiles.d/tmp.conf `配置查看清理的临时文件规则。
```bash
> cat /usr/lib/tmpfiles.d/tmp.conf
D /tmp 1777 root root -
```

如上图设置的`-`规则，则会在每次重启时自动清理。

如果需要手动清理临时文件，可以使用以下命令：

```bash
# 先“测试”哪些文件会被删除（无实际操作，仅列出）
find /tmp -type f -mtime +7 -print
# 确认无误后，执行删除
sudo find /tmp -type f -mtime +7 -delete
```
## 删除snap软件包

snap作为Ubuntu系统里臭名昭著的软件包管理器，如不需要其自带的一些软件，也可以卸载。

> 以下部分参考了[Linux中国的博客](https://zhuanlan.zhihu.com/p/511438456)

注： 这些步骤在``Ubuntu 22.04 LTS Jammy Jellyfish``中进行了测试。然而，它应该也适用于所有的 Ubuntu 系统版本。但是有些网友经实测发现，仍然会下载snap

首先需要说明：snap包管理系统安装了Ubuntu中的两个重要应用：软件商店和Firefox，所以，如果要删除snap包管理系统，需要先评估你是否需要这两个软件。

先通过`snap list`列出所有的snap包：

```bash
sudo snap remove --purge firefox
sudo snap remove --purge snap-store
sudo snap remove --purge gnome-3-38-2004
...other snap packages...
```

然后，移除snap服务

```bash
sudo apt remove --autoremove snapd
```

但是，即使使用上述命令移除了因为apt触发器仍然存在，`sudo apt update` 命令会再一次将 Snap 安装回来。

我们需要在 /etc/apt/preferences.d/ 目录下创建一个 apt 设置文件 nosnap.pref 来关闭 Snap 服务。
```bash
sudo gedit /etc/apt/preferences.d/nosnap.pref
```

在文件中添加以下内容：

```bash
Package: *
Pin: release a=*
Pin-Priority: -10
```

如果后续需要再次安装snap，可以运行以下命令：

```bash
sudo rm /etc/apt/preferences.d/nosnap.pref
sudo apt update && sudo apt upgrade
sudo snap install snap-store
sudo apt install firefox
```