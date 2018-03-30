> 以下内容年代久远，内容的正确性不能保证。

本来以为买个 openwrt 的路由器随便配置一下就 OK 了，后来发现真的全是坑。本文诣在介绍在小米路由器 mini 配置 Shadowsocks 的实现过程，不介绍具体原理。

## 开始
准备稳定版的小米路由器 mini 、 U 盘（ FAT/FAT32 格式）
## 路由器刷机
### 刷官方开发版固件
- 下载开发版 ROM 包(`miwifi.bin`)复制到 U 盘的根目录；
- 断开小米路由器的电源，将 U 盘插入路由器USB接口；
- 用细长工具将`reset`键按下，重新接入电源，等待指示灯变为黄色闪烁状态后松开`reset`键；
- 等待刷机完成，整个过程约为 10 分钟，完成后系统会自动重启并进入自动启动状态；如果出现异常、失败、 U 盘无法读取的状况，会进入红灯状态，建议重试或更换 U 盘再试。
### 开启 ssh
- 在[小米官方](https://d.miwifi.com/rom/ssh)下载`ssh`工具并记住`root`密码;
- 将下载的`bin`工具包放在 U 盘的根目录，不要改文件名（文件名为`miwifi_ssh.bin`）;
- 断开小米路由器的电源，将U盘插入USB接口
- **按住reset按钮之后重新接入电源**，指示灯变为黄色闪烁状态即可松开reset键；
- 等待3-5秒后安装完成之后，小米路由器会自动重启。
### 刷入 openwrt
我刷的固件版本是[PandoraBox-ralink-mt7620-xiaomi-mini-squashfs-sysupgrade-r512-20150309.bin](http://downloads.openwrt.org.cn/PandoraBox/Xiaomi-Mini-R1CM/stable/PandoraBox-ralink-mt7620-xiaomi-mini-squashfs-sysupgrade-r512-20150309.bin)
本地下载之后放在`/tmp`下执行刷机命令：
~~~
mtd -r write /tmp/PandoraBox-ralink-mt7620-xiaomi-mini-squashfs-sysupgrade-r512-20150309.bin OS1
~~~

## 删除固件中的软件包

原有固件包的一些内容是不能正常使用的，要删掉重装。
修改`/etc/opkg.conf`文件，可以通过路由器后台管理界面直接修改：
系统 > 软件包 > 配置
~~~
arch all 100
arch ramips_24kec 200
arch ramips 300
arch mips 400
arch unkown 500

src/gz barrier_breaker_base http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/base
src/gz barrier_breaker_luci http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/luci
src/gz barrier_breaker_management http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/management
src/gz barrier_breaker_oldpackages http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/oldpackages
src/gz barrier_breaker_packages http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/packages
src/gz barrier_breaker_routing http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/routing
src/gz barrier_breaker_telephony http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/telephony
~~~
使用未失效的更新源：
~~~
src/gz openwrt_dist http://openwrt-dist.sourceforge.net/releases/ar71xx/packages
src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/releases/luci/packages
~~~

~~opkg remove luci-app-shadowsocks~~

~~opkg remove shadowsocks-libev~~

~~opkg remove luci-app-chinadns~~

~~opkg remove ChinaDNS-C~~
## 安装 shadowsocks-libev-spec

下载地址
~~~
http://sourceforge.net/projects/openwrt-dist/files/luci-app/shadowsocks-spec/luci-app-shadowsocks-spec_1.3.2-1_all.ipk/download

http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/2.1.4-87ec497/ramips/shadowsocks-libev-spec_2.1.4-1_ramips_24kec.ipk/download
~~~

换源 libc 需要手动安装

~~~
cd /tmp
wget http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/base/libc_0.9.33.2-1_ramips_24kec.ipk
wget http://mirrors.ustc.edu.cn/openwrt/barrier_breaker/14.07/ramips/mt7620a/packages/base/libc_0.9.33.2-1_ramips_24kec.ipk
~~~

## 配置dnsmasq.d

~~~
cd /etc
mkdir dnsmasq.d
cd dnsmasq.d
wget http://www.ilucong.net/file/Black_List.conf
~~~
该链接已经失效，通过[archive.org](archive.org)勉强翻出来了……
编辑/etc/dnsmasq.conf文件
在文本底部添加
~~~
conf-dir=/etc/dnsmasq.d
~~~
## 重启 dnsmasq 服务的命令
~~~
/etc/init.d/dnsmasq restart
~~~
## opkg update
更新一下
~~~
opkg update
~~~
安装 libc
```
opkg install libc_0.9.33.2-1_ramips_24kec.ipk
```
安装 shadowsocks
~~~
cd /tmp
opkg install shadowsocks-libev-spec_2.1.4-1_ramips_24kec.ipk
opkg install luci-app-shadowsocks-spec_1.3.2-1_all.ipk
~~~
## 配置 Shadowsocks
openwrt 后台菜单里面设置，一目了然。
`代理方式`下拉菜单，选择全局代理;`代理协议` 选 `TCP+UDP`；
`访问控制` > `LAN` 下拉菜单选择白名单，填入 `1.1.1.1`；
DNCP/DNS中DNS转发到 `127.0.0.1:5300` ，并且在HOST和解析文件中 忽略解析文件。
## 添加自定义防火墙规则
网络 > 防火墙 > 自定义规则
~~~
arch all 100
arch noarch 200
arch ralink 300
arch ramips_24kec 400

ipset -N redir iphash
iptables -t nat -A PREROUTING -p tcp -m set --match-set redir dst -j REDIRECT --to-port 1080
~~~
## 自定义修改黑名单
比如说在`dnsmasq.d`文件中加入域名`cdninstagram.com`。
## 完成
之后某天又一次不小心重置了路由器，又折腾了一天之久，弄多了也觉得烦，之后就不想再去折腾了。总之能用就不要动了。
某天 Shadowsocks 服务和 goAgent 一样被干掉了再另想办法。
## 参考资料
- [相关项目](https://github.com/shadowsocks/openwrt-shadowsocks/releases)
- [小米 Mini OpenWRT 下配置 shadowsocks](http://www.hopol.cn/2015/05/245/)
- [小米 mini openwrt 源](http://www.hopol.cn/2015/05/236/)
- [小米路由器mini——从稳定版到openwrt](https://menyifan.com/2015/10/06/mini1/)
- [小米路由器mini折腾之配置opkg篇](https://blog.phpgao.com/xiaomi_router_opkg.html)(对应的反向代理地址已经过时)
- [china-dns+s-s最新懒人智能兲朝上网姿势](http://www.right.com.cn/forum/forum.php?mod=viewthread&tid=160297&page=1#pid999973)