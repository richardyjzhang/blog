# 使用 fail2ban 保护 SSH 端口

## 背景

暴露在公网上的服务器，从日志中可以看出来每天都有很多机器人在进行扫描，尝试 SSH 登录。这些大部分都是用字典来探测弱密码，虽然高强度的口令可以有效防止脚本的扫描，但如果能屏蔽一些扫描，总归是更安全的。

[fail2ban](https://github.com/fail2ban/fail2ban)，官网介绍为`ban hosts that cause multiple authentication errors`，很直观，就是对于多次尝试失败的主机进行屏蔽。

![fail2ban](./fail2ban.png)

fail2ban 的原理是，不断扫描`/var/log/auth.log`等日志文件，找到多次失效的 IP 地址，并动态设置系统防火墙规则，实现对这些 IP 的屏蔽。

## 安装

在我的 Rocky Linux 主机上，fail2ban 在`epel-release`仓库里，其他发行版看其他人的文章，也可以通过 apt 等包管理工具来安装。

```shell
sudo dnf install fail2ban
```

## 配置

fail2ban 的配置文件位于`/etc/fail2ban/`路径中，其中`jail.conf`定义了拦截规则。官方手册建议，如果要自定义拦截规则，应该创建`jail.local`文件，而不是直接在`jail.conf`上修改，这样可以避免在 fail2ban 升级时，破坏了我们修改的配置。

```shell
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

配置文件分为若干区段，为了实现我们保护 SSH 端口的目标，需要关注 Default 区段和 sshd 区段，需要重点关注的配置参数如下：

```/etc/fail2ban/jail.local
# 仅标注了需要注意和修改的配置项，其他未列出
# 保留了部分字段的英文注释（人家写得挺好的）

[Default]
# "bantime" is the amount of time that a host is banned, integer in seconds or
# time abbreviation format (m - minutes, h - hours, d - days, w - weeks, mo - months, y - years).
bantime = 10m

# A host is banned if it has generated "maxretry" during the last "findtime"
findtime = 10m

# "maxretry" is the number of failures before a host get banned.
maxretry = 5

# "enabled" enables the jails.
#  By default all jails are disabled, and it should stay this way.
#  Enable only relevant to your setup jails in your .local or jail.d/*.conf
#
# true:  jail will be enabled and log files will get monitored for changes
# false: jail is not enabled
enabled = false

[sshd]
enabled = true
port = 10086
bantime = 30m
findtime = 5m
maxretry = 3
```

上面的配置里，默认规则是 10 分钟触发 5 次就屏蔽 10 分钟；我针对 SSH 端口（我的服务器使用 10086）修改了配置，5 分钟内触发 3 次就屏蔽半个小时。

修改配置文件后，类似于 nginx，可以使用`fail2ban-client -t`来测试，以免出现一些拼写或语法错误。

可以使用 systemd 开启服务，并通过手机/笔记本电脑等方式尝试错误登录来验证。

```shell
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
sudo fail2ban-client status sshd
```

至此，已经可以实现对 SSH 端口比较好的保护。（然而，防呆不防傻，你要非设置 123456 这种密码，啥技术也救不了，所以关键还是设置比较强的口令。）fail2ban 的功能还有很多，感兴趣可以自己了解。
