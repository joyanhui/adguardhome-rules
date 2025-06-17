# 本仓库旨在提供adguardhome的规则

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

### adguardhome替代路由器自带的Dnsmasq

- 关闭Dnsmasq的dns功能：openwrt web界面：网络>DHCP/DNS>高级设置 `DNS 服务器端口` 默认是53修改为0 即可关闭
- 修改dhcp下发的dns ：openwrt web界面：网络>接口 找到`br-lan` 编辑 一般在这个接口的DHCP服务器的高级设置里面 找到 `DHCP 选项` 添加或者修改一协调`6,192.168.1.1,223.6.6.6` 即给dhcp客户端分配两个dns地址192.168.1.1,223.6.6.6 如果你的 adguardhome的安装ip不是在192.168.1.1那么这里要修改一下

### 基本配置

访问你运行adguardhome的加端口号 3000，例如:http://192.168.1.1:3000

dns端口就用53,因为上面我们已经把Dnsmasq的dns功能关闭了，所以这里adguard可以站哟机构53端口。 web访问端口不要用80,还是3000就好啦。

以及配置好管理账号密码以后 进入adg的管理面板

#### 基本dns配置

adg的管理面板：设置 > DNS设置

- 上游 DNS 服务器 随便输入一个可用的，不用配置太多，因为后面我们要用配置路径的方式用文件来配置上游dns规则。建议就输入一个 `223.5.5.5`
- 负载均衡/并行请求/最快ip地址 建议选择 并行请求
- Bootstrap DNS 服务器 输入几个公共dns的ip 例如 `123.125.81.6` `223.6.6.6` `119.29.29.29`一行一个
- DNS 服务配置 速度限制 建议修改20-100左右，我这里配置为50
- DNS缓存配置 根据自己需求和硬件性能配置我这里配置 缓存大小`1024000` 最小ttl`30` 最大ttl`1800`

> 此时 整个局域网的，包括路由器openwrt自己的dns 均由adguard接管。测试未访问过的网站是否可以正常ping到ip地址。例如：`ping news.163.com` `ping news.baidu.com` `ping image.baidu.com`

### 配置分流规则

#### 思路

adguard home支持 `[/example.local/]94.140.14.140: 指定为特定域名的上游服务器；` 这样的规则，那么我们只需要整理对应的域名列表 然后指定对应的上游dns即可。但是在adguard home的管理面部中的上游dns配置这个页面 显然不可能输入上几万个域名。但是adguard支持直接用文件配置。而我们又可以从Loyalsoldier等大佬的仓库获得一些我们需要的域名清单。虽然格式不符合要求，但是我们可以自己用程序稍微处理一下让他符合adguard home的格式，再把多个域名清单列表合并到一个里面那就大功告成了。

所以我提供好了 每天会更新的规则文件，你只需要下载回去自己再用脚本修改一下即可。 然后这个脚本 可以用wget下载sed修改，然后添加到计划人物每天运行就可以自动更新了。

####

#### 无辅助工具的情况下

你要确定你有办法可以直接方法github的文件，以及有无污染的dns上游可用。

建议自建一个github镜像和自建一个私有doh

建议准备一个域名，可以申请免费那些域名。不过不建议，还是自己从阿里或者cloudflare购买一个。8位数字的xyz域名一年只要几块钱。然后dns托管到cloudflare,开通 worker

- 配置自己的 github代理镜像网址 worker源码 https://github.com/joyanhui/gh-proxy/blob/master/index.js
- 配置自己的 doh源码：cfworks_doh.js

### doh 测试命令 大全

```sh

curl -H 'accept: application/dns-json' 'https://doh.你的.com:80443/dns-query?name=google.com&type=A'

dnslookup x.com  https://doh.你的的cf.com/dns
doggo x.com A AAAA MX  --time @https://doh.你的的cf.com/dns

dnslookup x.com  https://doh.你的.com:80443/dns-query
doggo x.com   @https://doh.le你的iyanhui.com:80443/dns-query

doggo x.com @https://cloudflare-dns.com/dns-query

```

## 广告过滤规则

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
