---
title: 魔改版TCP加速算法BBR安装脚本:Debian_Ubuntu（收集类）（转载）
date: 2017-12-05 14:53:00
categories:
	- Linux
tags:
	- BBR
	- 转载
	- 收藏
---

> 原文地址: https://blog.liyuans.com/archives/altered-bbr-onekey-script-collection.html
> 这里我仅做一个文档保存备份

# 雨落无声版本脚本

> 来源[雨落无声：魔改版 BBR 一键安装脚本 For Debian8 / Ubuntu16 +](http://www.hostloc.com/thread-372361-1-1.html)

注意：内核固定切换为 4.10.15，成功率高。

## 简介

- 在 Debian 8 和 Ubuntu16 + 系统上一键部署魔改版 BBR，自动换内核成 4.10.15 ，自动安装 Headers。
- 用户只需要将系统安装成 Debian 8 或者 Ubuntu 16 即可，剩下的交给脚本来吧。

## 提醒

**部分商家的 VPS 可能会遇到换内核之后无法启动系统的情况，所以请运行脚本前一定要备份好重要数据！！**

## 一键脚本

```bash
wget -N --no-check-certificate https://raw.githubusercontent.com/FunctionClub/YankeeBBR/master/bbr.sh && bash bbr.sh install
```

- 然后根据提示重启系统。重启完成后，运行

```bash
bash bbr.sh start
```

- 查看魔改 BBR 状态，运行下面的命令，如果看到有 tsunami 就表示开启成功！

```bash
sysctl net.ipv4.tcp_available_congestion_control
```

<!-- more -->

# Vicer 版本脚本

> 来源萌咖：[Debian/Ubuntu TCP BBR 改进版 / 增强版](https://moeclub.org/2017/06/24/278/)

注意：内核自动获取最新版，但是可能失败。

## 一键脚本

- 注意: 执行此命令会自动重启
本脚本在 Debian8，Ubuntu16.04 上通过测试

```bash
wget --no-check-certificate -qO 'BBR_POWERED.sh' 'https://moeclub.org/attachment/LinuxShell/BBR_POWERED.sh' && chmod a+x BBR_POWERED.sh && bash BBR_POWERED.sh
```

### 说明:

- 执行过程中会重新编译模块
- 模块默认为开机自动加载
- 模块名称: `tcp_bbr_powered`
- 可用 `modprobe tcp_bbr_powered` 命令进行加载模块
- 可执行 `lsmod |grep 'bbr_powered'`
- 结果不为空, 则加载模块成功
- 可执行 `sysctl -w net.ipv4.tcp_congestion_control=bbr_powered` 使用此模块

### 注意:

- 如遇报错: `Error! Header not be matched by Linux Kernel`，请用使用本博客提供的脚本重新开启 BBR
- 如遇报错: `Error! Install gcc-4.9`，首先尝试 `apt-get update`，再次执行此脚本
- 如果未解决，请自行想办法安装 `gcc-4.9`，或切换系统后再试

# 91 云一键脚本

> 来源 91yun: [修改版 BBR 安装，转载自 hostloc @Yankee](https://www.91yun.org/archives/16781)

**仅支持 Debian 8 & Ubuntu 16.04**

```bash
wget https://raw.githubusercontent.com/singhigh/502newbbr/master/502newbbr.sh
chmod +x 502newbbr.sh
./502newbbr.sh
```

# cxcool 版本（针对老系统）

> 来源 Hostloc: [[萌新教程] 如何在 Ubuntu 14.04 和 Debain 7 老系统上装 BBR 魔改版](http://www.hostloc.com/thread-372335-1-1.html)

- 给萌新和渣新指点一下如何 上 `TSUNAMI BBR`
只有 `DEBIAN 7` 和 `UBUNTU 14.04` 的地方应该怎么办呢 ？ 请看如下 。。。

- 开 BBR 的教程表示只要装 内核的 image 即可 ，header 没要求 。。然后魔改版的需要 。。
那应该怎么办呢 -3-。。。

- 还是到老地方，选择你 安装的对应内核 - -。。WHAT？ 对应内核找不到了 - - 输入： `uname -r`
查找内核地址：http://kernel.ubuntu.com/~kernel-ppa/mainline/
或者自己谷歌：site:centos.org 内核版本

- 以这个 4.10.15 内核为例：
```bash
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10.15/linux-headers-4.10.15-041015_4.10.15-041015.201705080411_all.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10.15/linux-headers-4.10.15-041015-generic_4.10.15-041015.201705080411_amd64.deb
dpkg -i linux-headers-4.10.*.deb    ---- 这里你是 4.10.xx 就这么写 如果是 4.11 和 4.12 那么自己改别无脑复制 -3-
```

- 好了对应的内核装好之后, 老系统要装 `gcc 4.9 +` 很多新人可能卡在这个步骤 。。。
老的系统通常 `ubuntu` 默认的 `apt-get` 库里没 `4.9 +` 或者就是你装了还是识别不到 这种奇葩问题 。。。那么接下来就来解决：
先是 UBUNTU 14.04 安装方法
```bash
apt-get update
apt-get install software-properties-common
add-apt-repository ppa:ubuntu-toolchain-r/test              ----这一步可能提示你要按个回车 继续安装。。
apt-get update
apt-get upgrade
apt-get install gcc-4.9 g++-4.9
updatedb
ldconfig
locate gcc
```

- 再是 `DEBIAN7`：
`apt-get update`
`apt-get upgrade`
`nano /etc/apt/sources.list`    ---把里面的`wheezy` 这个词全部替换成` jessie ,CTRL O ,CTRL X` 退出。
`apt-get update`
`apt-get install gcc-4.9 g++-4.9`
`nano /etc/apt/sources.list`   ---把`jessie` 换回`wheezy`

- 这些步骤好了之后 gcc -v ，如果你发觉他显示的 gcc 版本还是 4.8X 的话 。继续如下步骤，注意一行一行的复制 。。回车进行下一行，当最后一行输入完毕之后会提示你 gcc 安装位置更新成功 - -。。
```bash
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.9 49 \
--slave /usr/bin/g++ g++ /usr/bin/g++-4.9 \
--slave /usr/bin/gcc-ar gcc-ar /usr/bin/gcc-ar-4.9 \
--slave /usr/bin/gcc-nm gcc-nm /usr/bin/gcc-nm-4.9 \
--slave /usr/bin/gcc-ranlib gcc-ranlib /usr/bin/gcc-ranlib-4.9
```

- 那么再来一次 `gcc -v` 看到是 4.9x 之后你就可以愉快的进行安装了 。。。
  安装完毕之后，附上小白专用 `sysctl` 预设置文件。。自己用 `SSHFTP` 上传到 `/etc/` 目录下 。。替代原有文件。。`sysctl.conf (2.64 KB)`

- 然后来一发 `sysctl -p`
OK 了 ，保存好了 - - 重启也不会掉存档了 。。。。但是模块不会自己启动，等我过个 1-2 天再更新傻瓜教程。。。。
于是这个很 LOW 的 菜鸟专用教程结束了 -3-。。。
再次给大佬跪安

## 原帖 @Yankee

> 来源 Hostloc：[魔改版 BBR 一键安装脚本 For Debian8 / Ubuntu16 +](http://www.hostloc.com/thread-372277-1-2.html)

- [初版 BBR](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/plain/net/ipv4/tcp_bbr.c)
- [魔改 BBR](https://gist.github.com/anonymous/ba338038e799eafbba173215153a7f3a/raw/55ff1e45c97b46f12261e07ca07633a9922ad55d/tcp_tsunami.c)

### 注意

- 编译时系统必须安装 `4.10` 以上版本的 kernel 及对应的 `linux-header`，`gcc` 版本应在 `4.9` 以上
以 `4.10.9` 为例，需先更换内核，再先后安装：
（http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10.9/）
`linux-headers-4.10.9-041009_4.10.9-041009.201704080516_all.deb`
`linux-headers-4.10.9-041009-generic_4.10.9-041009.201704080516_amd64.deb`

### 安装

- 以下步骤在 debian 8 及 ubuntu 16 上测试通过：
```bash
apt-get install make gcc-4.9
wget -O ./tcp_tsunami.c https://gist.github.com/anonymous/ba338038e799eafbba173215153a7f3a/raw/55ff1e45c97b46f12261e07ca07633a9922ad55d/tcp_tsunami.c
echo "obj-m:=tcp_tsunami.o" > Makefile
make -C /lib/modules/$(uname -r)/build M=`pwd` modules CC=/usr/bin/gcc-4.9
install tcp_tsunami.ko /lib/modules/$(uname -r)/kernel
depmod -a
insmod tcp_tsunami.ko
sysctl -w net.ipv4.tcp_congestion_control=tsunami
```

- 关键参数：`bbr_bw_rtts`, `bbr_min_rtt_win_sec`, `bbr_probe_rtt_mode_ms`, `bbr_cwnd_min_target`, `bbr_drain_gain`
- 关键数组：`bbr_pacing_gain`

经过一个月的测试，魔改 BBR 比起正常版的 BBR 确实强上不少，最起码以前看 y2b 4k 断断续续的远东 ss 现在能流畅播放了。
以上版本的 BBR 不保证普适性，建议自行修改参数测试

- 参考链接：
	1. http://blog.csdn.net/dog250/article/details/52939004
	2. http://blog.csdn.net/dog250/article/details/52879298
	3. http://blog.csdn.net/dog250/article/details/52972502
	4. https://patchwork.ozlabs.org/patch/671069/

### 补充

- 关于自启动 (optional)
由于自定义的模块没有直接编译入 debian 内核，现阶段系统重启后必须动态加载模块，并重新设定拥塞控制算法，建议将启动指令写入 startup，范例如下：
```bash
PATH_EXEC="/etc/init.d/tsunami_up"
cat>>$PATH_EXEC<'EOF'
modprobe tcp_tsunami
sysctl -w net.ipv4.tcp_congestion_control=tsunami
exit 0
EOF
sudo chmod +x $PATH_EXEC
echo -e "`cat /etc/rc.local|grep -v -E '($PATH_EXEC|exit 0)'`\n$PATH_EXEC\n\nexit 0" > /etc/rc.local
$PATH_EXEC
```

- Additional Optimization (recommended)
参考：
https://github.com/shadowsocks/shadowsocks/wiki/Optimizing-Shadowsocks
https://github.com/breakwa11/shadowsocks-rss/wiki/ulimit

- Experimental Optimization (optional)
实验性调优，通过对 fq 参数的进阶设定，以期提高 BBR 的发包能力上限
https://www.systutorials.com/docs/linux/man/8-tc-fq/





