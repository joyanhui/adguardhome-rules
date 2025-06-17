# 本仓库旨在提供adguardhome的规则

> 使用问题交流请去恩山或[discussions](https://github.com/joyanhui/adguardhome-rules/discussions)

> bug反馈和建议以及公共doh/github proxy分享都可以提[issues](https://github.com/joyanhui/adguardhome-rules/issues)，我会尽快处理.给个star我会很开心.

> PR请尽量通过一次commit提交过来.

> 关于爱快的分流规则请查看我另外的仓库 [[joyanhui/ikuai-bypass]](https://github.com/joyanhui/ikuai-bypass)

## 单adguardhome分流解析

### 安装和启动

这里以opwenwrt主路由的情况为例子，其他环境随便使用docker或者官方的安装命令等都可以正常安装。此文不涵盖相关内容。

非常不建议使用luci-app-adguardhome,不但没必要而且会带来更多困扰。

先确定你的openwrt配置了正确的源，并且没有项目展会用3000端口，然后依次运行下面两行命令

```sh
opkg update #更新软件列表
opkg install adguardhome #安装
/etc/init.d/adguardhome start #启动
/etc/init.d/adguardhome enable #下次开机自动运行
```

### adguardhome 替代路由器自带的Dnsmasq

- 关闭Dnsmasq的dns功能：openwrt web界面：网络>DHCP/DNS>高级设置 `DNS 服务器端口` 默认是53修改为0 即可关闭
- 修改dhcp下发的dns ：openwrt web界面：网络>接口 找到`br-lan` 编辑 一般在这个接口的DHCP服务器的高级设置里面 找到 `DHCP 选项` 添加或者修改一协调`6,192.168.1.1,223.6.6.6` 即给dhcp客户端分配两个dns地址192.168.1.1,223.6.6.6 如果你的 adguardhome的安装ip不是在192.168.1.1那么这里要修改一下

> 如果是爱快直接使用dns代理功能指定到adguardhome即可

### adguardhome 基本配置

访问你运行adguardhome的加端口号 3000，例如:http://192.168.1.1:3000

dns端口就用53,因为上面我们已经把Dnsmasq的dns功能关闭了，所以这里adguard可以站哟机构53端口。 web访问端口不要用80,还是3000就好啦。

以及配置好管理账号密码以后 进入adg的管理面板

#### adguardhome 基本dns配置

adg的管理面板：设置 > DNS设置

- 上游 DNS 服务器 随便输入一个可用的，不用配置太多，因为后面我们要用配置路径的方式用文件来配置上游dns规则。建议就输入一个 `223.5.5.5`
- 负载均衡/并行请求/最快ip地址 建议选择 并行请求
- Bootstrap DNS 服务器 输入几个公共dns的ip 例如 `123.125.81.6` `223.6.6.6` `119.29.29.29`一行一个
- DNS 服务配置 速度限制 建议修改20-100左右，我这里配置为50
- DNS缓存配置 根据自己需求和硬件性能配置我这里配置 缓存大小`1024000` 最小ttl`30` 最大ttl`1800`

> 此时 整个局域网的，包括路由器openwrt自己的dns 均由adguard接管。测试未访问过的网站是否可以正常ping到ip地址。例如：`ping news.163.com` `ping news.baidu.com` `ping image.baidu.com`

### adguardhome 配置分流规则

#### adguardhome 分流思路

adguard home支持 `[/example.local/]94.140.14.140: 指定为特定域名的上游服务器；` 这样的规则，那么我们只需要整理对应的域名列表 然后指定对应的上游dns即可。但是在adguard home的管理面部中的上游dns配置这个页面 显然不可能输入上几万个域名。但是adguard支持直接用文件配置。而我们又可以从Loyalsoldier等大佬的仓库获得一些我们需要的域名清单。虽然格式不符合要求，但是我们可以自己用程序稍微处理一下让他符合adguard home的格式，再把多个域名清单列表合并到一个里面那就大功告成了。

所以我提供好了 每天会更新的规则文件，你只需要下载回去自己再用脚本修改一下即可。 然后这个脚本 可以用wget下载sed修改，然后添加到计划人物每天运行就可以自动更新了。 地址是 `https://github.com/joyanhui/adguardhome-rules/tree/release_file`

#### 无辅助上网环境的情况下

你要确定你有办法可以直接方法github的文件，以及有无污染的dns上游可用，建议自己准备一个域名自己从cloudflare自建一个github镜像站和自建一个私有doh.

域名，可以申请免费那些域名。不过建议从阿里云或者cloudflare或者其他地方购买一个。8位数字的xyz域名一年只要几块钱。免费的二级域名比较容易在敏感时期全部被大局域网屏蔽，然后dns托管到cloudflare,开通 cloudflare worker。这个网络很多教程。

- 配置自己的 github代理镜像网址 worker源码 https://github.com/joyanhui/gh-proxy/blob/master/index.js
- 配置自己的 doh源码： [cfworks_doh.js](https://github.com/joyanhui/adguardhome-rules/blob/main/cfworks_doh.js) 记得修改里面的?dns-query=

如果你不想折腾可以从网络搜索其他人的，这里提供几个目前可用的

##### github proxy 地址收集

- https://goppx.com/
- github proxy: https://github.akams.cn/
- github proxy: https://ghproxy.net/

##### 无污染的 dns 地址收集

dot因为端口特殊的问题基本都不稳定，doh的话 cloudclare在多数下可用

- https://dns.cloudflare.com/dns-query

#### 创建一个自动更新adguard上游dns规则文件的脚本

创建一个自动更新脚本 比如

`nano /overlay/data/adguard_upstream_dns_file_update.sh` 内容如下

```sh
#!/bin/sh
rm -rf /overlay/data/adguard_upstream_dns_file.txt
wget "https://你的githubProxy代理站地址/https://raw.githubusercontent.com/joyanhui/adguardhome-rules/refs/heads/release_file/ADG_chinaDirect_WinUpdate_Gfw.txt" -O /overlay/data/adguard_upstream_dns_file.txt

sed -i 's/d-o-h.you-cf-domain.com/你的doh域名部分/g' /overlay/data/adguard_upstream_dns_file.txt
sed -i 's/your-suffix/你的doh后缀部分/g' /overlay/data/adguard_upstream_dns_file.txt

/etc/init.d/adguardhome restart
```

如果你没有自己的私有githubPoryx和doh

- `https://你的githubProxy代理站地址/` 修改为 `https://goppx.com/`
- `你的doh域名部分` 可以修改为 `dns.cloudflare.com` `你的doh后缀部分` 修改为`dns-query`

[[joyanhui/adguardhome-rules/tree/release_file]](https://github.com/joyanhui/adguardhome-rules/tree/release_file) 分支中有多个每日自动更新的规则文件，选择其中一个适合你的

你需要根据你的情况修改一下 里面两个地址 以及文件路径。如果不是openwrt还需要修改对应的重启adgurd的命令

运行一次 `sh /overlay/data/adguard_upstream_dns_file_update.sh`,然后自己检查获取到的adguard_upstream_dns_file.txt这个文件是否正常。

如果你中间运行出错了，导致dns全部失效 可以执行下面命令恢复dns基本的运行

```sh
echo '114.114.114.114
223.6.6.6
'>/overlay/data/adguard_upstream_dns_file.txt
```

### 添加到计划任务

```sh
chmod +x  /overlay/data/adguard_upstream_dns_file_update.sh
```

然后在openwrt的web管理里面：系统>计划任务 增加一行

```sh
0 12 * * *   /overlay/data/adguard_upstream_dns_file_update.sh
```

#### 配置adguard home 的上游dns为规则文件模式

找到你的adguardhome的yaml配置文件。上面按照的是在`/etc/adguardhome.yaml`,如果你不确定可以搜索一下 `find / -name "*uard*ome.yaml"`

编辑这个文件 找到这行 `upstream_dns_file` 修改为

```yaml
upstream_dns_file: /overlay/data/adguard_upstream_dns_file.txt
```

在修改`upstream_dns_file`并重启adguardhome后`upstream_dns` 部分的配置会失效，adguardhome的网页端也可以看到

在修改upstream_dns_file 之前可以先ping一个gfw的域名。例如 `ping google.com`,修改完成后 重启adguardhome ，然后再ping一下看看能否获取正确的ip

### 进阶 adguardhome 搭建自己的doh

本地的adguardhome也可以对外提供 doh。有两种方式 一种是自签证书[[在线生成]](https://www.toolhelper.cn/SSL/SSLGenerate)或者购买一个长期证书。另外一种方式就是配合证书管理和反向代理工具，下面用 lucky来做简单说明。

#### adguardhome 启用doh

在 adguardhome 的设置 加密设置里面 启用加密 HTTPS 端口 配置为一个可用的端口，例如 30443 不要打开https重定向，无意义

DNS-over-TLS 端口 和 DNS-over-QUIC 端口 因为牵扯到证书续签的问题，这里就干脆设置为0,不使用了。

然后 证书内容 粘贴一下随便一个目前可用的域名证书（暂时可用的就ok） 可以从lucky的ssl管理的 映射路径 里面 crt 和key文件里面获取到。

然后保存修改

#### lucky接管 adguardhome的https

lucky web服务正常可用的情况下，添加一个子规则：

- 前端地址 你的域名
- 后端地址 adguardhome 的https地址 例如 https://192.168.1.1:30443
- 忽略后端TLS证书验证 一定要勾选 这样我们就不用关心adguardhome的证书是否可用。

#### doh 测试命令 大全

```sh

dnslookup x.com  https://你的doh域名/你的doh后缀
doggo x.com A AAAA MX  --time @https://你的doh域名/你的doh后缀
doggo x.com  @https://你的doh域名/你的doh后缀

doggo x.com @https://cloudflare-dns.com/dns-query

```

## adguard 广告过滤规则 收集

adguard home广告过滤

```sh
# easylist国际网页
https://easylist-downloads.adblockplus.org/easylist.txt
# easylist国内网页
https://easylist-downloads.adblockplus.org/easylistchina.txt
# easylist-cookis 阻止Cookie标语，GDPR覆盖窗口和其他与隐私相关的通知
https://easylist-downloads.adblockplus.org/easylist-cookie.txt

# anti-AD中文区命中率最高 的广告过滤列表 电视盒子 日志收集、大数据统计
https://gh.leiyanhui.com/https://raw.githubusercontent.com/privacy-protection-tools/anti-AD/master/anti-ad-easylist.txt


```
