然后在路由器的dhcp/dns的 高级设置里面 DNS 服务器端口 修改成0 直接关闭

进入 192.168.1.1:3000网页不要改 依旧是3000,dns端口53直接用

进去 在设置 DNS设置

先添加一个上游dns 随便写个可用的就好，因为后面我们要做分流这里写了也没啥用了，现在是临时用一下。

```sh
223.5.5.5
114.114.114.114
```

并行请求注意下面要认真写Bootstrap DNS 服务器

```sh
123.125.81.6
223.6.6.6
119.29.29.29
1.1.1.1
8.8.8.8
```

然后测试 路由器的ssh 可以正常ping到域名后。

在“网络->接口”中找到br-lan，然后点击编辑。找到“DHCP服务器->高级选项”，找到“DHCP选项”，填入如下：

6,192.168.1.1,223.6.6.6

就是给客户的分配 dns的时候 分配 上面的地址的

这时候 adguard 已经接管了dns

很多人用 双 adguard 或者 mosdns/smartdns + adg 实在和luci-app-一样的搞笑

找到配置文件。这样按照的adg配置文件 是 `/etc/adguardhome.yaml`

找到配置项 `upstream_dns_file: ''` 通常在 `upstream_dns`下面修改为

```yaml
upstream_dns_file: '/overlay/data/adguard_upstream_dns_file.txt'
```

新建一个脚本

```sh
echo '
#!/bin/sh

wget  "https://gh.leiyanhui.com/https://github.com/Loyalsoldier/v2ray-rules-dat/raw/refs/heads/release/direct-list.txt" -O /overlay/data/adg_tmp_direct.txt
sed "s/^full://g;/^regexp:.*$/d;s/^/[\//g;s/$/\/]tls:\/\/dot.pub/g" -i /overlay/data/adg_tmp_direct.txt


wget  "https://gh.leiyanhui.com/https://github.com/Loyalsoldier/v2ray-rules-dat/raw/refs/heads/release/win-update.txt"  -O /overlay/data/adg_tmp_winupdate.txt
sed "s/^full://g;/^regexp:.*$/d;s/^/[\//g;s/$/\/]tls:\/\/dot.pub/g" -i /overlay/data/adg_tmp_winupdate.txt



wget  "https://gh.leiyanhui.com/https://github.com/Loyalsoldier/v2ray-rules-dat/raw/refs/heads/release/gfw.txt"  -O /overlay/data/adg_tmp_gfw.txt
sed "s/^full://g;/^regexp:.*$/d;s/^/[\//g;s/$/\/]https:\/\/doh.leiyanhui.com\/dns/g" -i /overlay/data/adg_tmp_gfw.txt

cat /overlay/data/adg_tmp_direct.txt > /overlay/data/adguard_upstream_dns_file.txt
echo "#-----" >> /overlay/data/adguard_upstream_dns_file.txt
cat /overlay/data/adg_tmp_winupdate.txt >>  /overlay/data/adguard_upstream_dns_file.txt
echo "#-----" >> /overlay/data/adguard_upstream_dns_file.txt
cat /overlay/data/adg_tmp_gfw.txt >>  /overlay/data/adguard_upstream_dns_file.txt


echo "#-----" >> /overlay/data/adguard_upstream_dns_file.txt
echo "https://public.dns.iij.jp/dns-query
https://doh.opendns.com/dns-query
https://dns.cloudflare.com/dns-query
https://dns.twnic.tw/dns-query
https://doh.leiyanhui.com/dns
" >> /overlay/data/adguard_upstream_dns_file.txt

rm -rf /overlay/data/adg_tmp_*.txt


/etc/init.d/adguardhome restart
' > /overlay/data/adguard_upstream_dns_file_update.sh

# 执行一次
chmod +x /overlay/data/adguard_upstream_dns_file_update.sh
sh /overlay/data/adguard_upstream_dns_file_update.sh
```

上面这段脚本的意思 是

- 删除本地的上游dns配置文件
- 通过github代理站从Loyalsoldier大神仓库拉一个国内域名白名单的域名列表文件临时替代刚刚删除的上游dns配置文件
- 给白名单的域名都添加一个指定的dns服务器，我这里使用的 tls://dot.pub(腾讯dnspod家的)
- 然后下载gfw的域名列表，都给他们加一个doh
- 最后添加几个 doh/dot的上游dns 给没有匹配的域名使用

## doh 启用

随便配置 配置一个域名证书，然后在 lucky里面反响代理 忽略证书错误就好了

测试

```sh

curl -H 'accept: application/dns-json' 'https://doh.jia.leiyanhui.com:50443/dns-query?name=google.com&type=A'


curl -v -k -H   'accept: application/dns-json' 'https://192.168.1.1:30443/dns-query?name=google.com&type=A'


dnslookup x.com  https://doh.leiyanhui.com/dns
doggo x.com A AAAA MX  --time @https://doh.leiyanhui.com/dns
doggo x.com   @https://doh.leiyanhui.com/dns

dnslookup x.com  https://doh.jia.leiyanhui.com:50443/dns-query
doggo x.com   @https://doh.jia.leiyanhui.com:50443/dns-query

dnslookup x.com  https://192.168.1.1:30443/dns-query


doggo example.com @https://cloudflare-dns.com/dns-query

```
