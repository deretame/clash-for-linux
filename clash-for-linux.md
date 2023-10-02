# [clash-for-linux](https://blog.iswiftai.com/posts/clash-linux/)

# Clash 下载

### 在 [Clash release](https://github.com/Dreamacro/clash/releases) 页面下载相应的版本

然后使用 `gunzip` 命令解压，并重命名为 `clash`：

```shell
gunzip clash-linux-amd64-v1.14.0.gz
mv clash-linux-amd64-v1.14.0 clash
```

为 clash 添加可执行权限：

```shell
chmod u+x clash
```

Clash 运行时需要 `Country.mmdb` 文件，当第一次启动 Clash 时（使用 `./clash` 命令） 会自动下载（会下载至 `/home/XXX/.config/clash` 文件夹下）。自动下载可能会因网络原因较慢，可以访问[该链接](https://github.com/Dreamacro/maxmind-geoip/releases)手动下载。

*`Country.mmdb` 文件利用 GeoIP2 服务能识别互联网用户的地点位置，以供规则分流时使用。*

## 配置文件

一般的网络服务提供了 Clash 订阅链接，可以直接下载链接指向的文件内容，保存到 `config.yaml` 中。或者使用订阅转换服务（如[该链接](https://converter.niallapi.top)。也可以自行搭建，可参考[该文章](https://blog.iswiftai.com/posts/docker-subscription-converter/)），将其它订阅转换为 Clash 订阅。

这里推荐使用订阅转换服务，转换后的配置文件已添加更为强大的分流规则。就可以将 Clash 一直保持后台运行，自动分流，且会自动选择最优节点。

**Clash 配置文件的完整参数介绍见[官方文档](https://dreamacro.github.io/clash/)。**

- 如果使用订阅转换服务，对于转换后的订阅链接，可以使用以下命令来下载配置文件：

```shell
curl -o config.yaml 'longURL'
```

- 对于 `suo.yt` 短链接，需要重定向，因此使用以下命令来下载配置文件：

```shell
curl -L -o config.yaml 'shortURL'
```

# Clash as a daemon

将 Clash 转变为系统服务，从而使得 Clash 实现常驻后台运行、开机自启动等。  
***普通用户需要 sudo 权限。***

## 配置 systemd 服务

Linux 系统使用 systemd 作为启动服务器管理机制，首先把 Clash 可执行文件拷贝到 `/usr/local/bin` 目录，相关配置拷贝到 `/etc/clash` 目录。

```shell
sudo mkdir /etc/clash
sudo cp clash /usr/local/bin
sudo cp config.yaml /etc/clash/
sudo cp Country.mmdb /etc/clash/
```

创建 systemd 服务配置文件 `sudo vim /etc/systemd/system/clash.service`：

```shell
[Unit]
Description=Clash daemon, A rule-based proxy in Go.
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target
```

## 使用 systemctl

使用以下命令，让 Clash 开机自启动：

```shell
sudo systemctl enable clash
```

然后开启 Clash：

```shell
sudo systemctl start clash
```

查看 Clash 日志：

```shell
sudo systemctl status clash
sudo journalctl -xe
```

## 使用代理

### 利用 Export 命令使用代理

Clash 运行后，其在后台监听某一端口。Ubuntu 下使用代理，需要 `export` 命令。根据 config 配置文可以查看到件Clash 代理端口（订阅转换后，端口为7890），设置系统代理命令为：

```shell
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

可以将该命令添加到 `.bashrc` 中，登陆后该用户自动开启代理。

取消系统代理：

```shell
unset  http_proxy  https_proxy  all_proxy
```

***一般下载数据集时，记得取消代理。***

## DashBoard 外部控制

外部控制端口为 9090，因此也可以访问[该链接](http://clash.razord.top/#/proxies)，输入 IP 地址（需本机可以访问的 IP）以及端口号 9090，来进入 Clash Dashboard 进行节点的选择。也可以在服务器自行搭建 Clash Dashboard，请参见[该项目](https://github.com/Dreamacro/clash-dashboard)。不过 Clash Dashboard 用处不大，使用**订阅转换**后的配置文件包含了自动选择的功能，Clash 会自动选择延迟最低的节点。

## 设置密码

`export` 命令其他用户执行后也可以使用该代理，此时通过可以更换代理端口、添加密码等措施加以限制。修改 `/etc/clash/config.yaml` 文件部分配置：

```shell
mixed-port: 12345
authentication:
  - "用户名1:密码1"
  - "用户名2:密码2"
allow-lan: true
mode: Rule
log-level: info
external-controller: :9090
```

`mixed-port: 12345` 就是混合代理端口，即使用代理时所指定的端口。然后需要重启 Clash，命令为：

```shell
export https_proxy=http://用户名1:密码1@127.0.0.1:12345 http_proxy=http://用户名1:密码1@127.0.0.1:12345 all_proxy=socks5://用户名1:密码1@127.0.0.1:12345
```

## TUN 模式

新版的 Clash Premium 内核支持 TUN 模式，且目前已支持 Linux 系统下的 `auto-route` 和 `auto-detect-interface`，无需手动设置转发表，可以方便快捷的实现 **透明网关（旁路由）** 的功能。

首先需要下载 [Clash Premium](https://github.com/Dreamacro/clash/releases/tag/premium) 版本，替换上面的 `clash` 文件。接着需要设置 Linux 系统，开启转发功能。编辑文件 `/etc/sysctl.conf`，添加以下内容：

```shell
net.ipv4.ip_forward=1
```

保存退出后，执行以下命令使修改生效：

```shell
保存退出后，执行以下命令使修改生效：
```

然后接着需要关闭系统的 DNS 服务，使用以下命令：

```shell
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

**关于代理环境下 DNS 解析行为的深入探讨，可以参见[浅谈在代理环境中的 DNS 解析行为](https://blog.skk.moe/post/what-happend-to-dns-in-proxy/)以及[我有特别的 DNS 配置和使用技巧](https://blog.skk.moe/post/i-have-my-unique-dns-setup/)。**  
接着需要设置 Clash 的配置文件，添加以下内容：

```shell
dns:
  enable: true
  listen: 0.0.0.0:53
  enhanced-mode: fake-ip
  nameserver:
    - 114.114.114.114
  fallback:
    - 8.8.8.8
tun:
  enable: true
  stack: system # or gvisor
  dns-hijack:
    - 8.8.8.8:53
    - tcp://8.8.8.8:53
    - any:53
    - tcp://any:53
  auto-route: true # auto set global route
  auto-detect-interface: true # conflict with interface-name
```

最后重启 Clash 服务即可，这样流量就会通过 TUN 接口转发，同时利用强大的分流规则，实现按需代理。也可以设置局域网内的网关地址和 DNS 服务器地址，实现透明网关。