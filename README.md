# EdnsPlus

智能DNS服务部署脚本，通过EDNS协议以获取更精准的查询结果，由Apad.Pro创建<br />
CentOS/Redhat已通过测试，其它x86_64的Linux系统可以参考源码修改后部署

## 产品介绍

EdnsPlus使用Overture+Dnsproxy+Socks2dns方案，配合脚本处理域名分流规则，可以更精准的解析域名，查询也更高效。支持UDP、TCP、DoH、DoT、DoQ方案，支持EDNS修改，支持Redis缓存，支持自定义规则解析，优化域名解析结果，拦截恶意域名，纯净无污染。

相比早期基于Chinadns根据中国大陆IP列表分流的方案，使用Overture前端根据IP分流后，会将原本使用境外DNS解析的域名优化为使用Dnsproxy智能分流：

- 默认使用GoogleDNS查询
- 域名存在境内CDN则使用境内DNS查询
- 恶意域名向AdGuardDNS查询实现屏蔽
- 自定义需要使用境外客户端IP解析的域名，模拟中国台湾用户的IP进行解析
- 当境外DNS查询结果为空时使用境内DNS再次查询，解决禁止境外解析且服务器未在境内的域名无法解析的问题

更多信息请查看 https://apad.pro/ednsplus/

## 部署服务

请务必将EdnsPlus部署在/usr/local/ednsplus目录，以免脚本无法使用

下载源码解压缩，再次提醒不要修改目录
```bash
wget https://mirror.apad.pro/dns/ednsplus.tar.gz
tar xzf ednsplus.tar.gz -C /usr/local
```

检测是否有端口冲突，端口包括：53 | 853 | 1053 | 5353 | 8053 | 9053 | 9153 | 9253
如有冲突请先关闭占用以下端口的程序后再执行下一步
```bash
netstat -tunlp | grep 53
```

安装服务脚本
```bash
cd /usr/local/ednsplus/tools
./install_service
```

更新解析规则
```bash
cd /usr/local/ednsplus/tools
./update_local
```

部署完成后，请开启防火墙的UDP53、TCP53、TCP80、TCP443端口，部分有硬件防火墙服务的云主机应在管理页面同时开启以上端口

## 如何开启Socks5代理查询模式

Socks5默认为未配置状态，在ecs_tw_domain添加域名可能会存在污染，强烈推荐开启sokcs5代理查询模式实现无污染查询，配置shadowsocks客户端后重启服务立即生效，请修改根目录下的shadowsocks2.env配置文件，依次填写远程shadowsocks服务端的加密方式、密码、IP、端口信息，正确填写后执行以下操作开启Socks5查询模式
```bash
cd /usr/local/ednsplus/tools
./restart_socks5
./update_socks5
```

## 如何关闭Socks5代理查询模式

使用本地查询更新规则更新即可
```bash
cd /usr/local/ednsplus/tools
./update_local
```


## 如何开启规则自动更新

使用crontab -e命令在最后一行添加对应规则即可

关闭Socks5的本地查询更新规则：
```bash
0 5 * * * /usr/local/ednsplus/tools/update_local
```

开启Socks5的代理查询更新规则：
```bash
0 5 * * * /usr/local/ednsplus/tools/update_socks5
```

## 如何自定义EDNS查询

部分域名可以通过自定义EDNS的方式加速，例如将github.com添加至ecs_tw_domain列表后，解析结果将会由原来的新加坡节点变为日本节点，大幅度提升访问速度
- 指定中国大陆EDNS查询，请添加域名至根目录下的ecs_cn_domain列表
- 模拟中国台湾EDNS查询，请添加域名至根目录下的ecs_tw_domain列表

添加完成后，请更新规则并重启DNS服务

关闭Socks5的本地查询更新重启：
```bash
cd /usr/local/ednsplus/tools
./update_local
./restart_dns
```

开启Socks5的代理查询更新重启：
```bash
cd /usr/local/ednsplus/tools
./update_socks5
./restart_dns
```

## 如何自定义hosts

请添加规则至根目录下的hosts列表后重启服务
```bash
cd /usr/local/ednsplus/tools
./restart_dns
```

## 如何开启Redis缓存

请先确认Redis服务在本机的6379端口正常运行，修改根目录下的config.yml配置文件，删除#cacheRedisUrl前的#号注释，重启DNS服务
```bash
cd /usr/local/ednsplus/tools
./restart_dns
```

## 如何开启DoH

DoH服务默认已开启，访问地址：
http://127.0.0.1:5353/dns-query

推荐使用nginx反代，如前端使用CDN加速，请关闭CC防御，此时建议nginx反代http，CDN回源80端口，减少TLS握手次数提升性能

nginx反代示例：
```bash
  server {
    ...
    location /dns-query {
	  set_real_ip_from 0.0.0.0/0;
      real_ip_header X-Forwarded-For;
      proxy_pass http://127.0.0.1:5353/dns-query;
    }
  }
```

## 如何开启与使用DoT/DoQ/8053端口

- 开启Dot/DoQ<br />
修改根目录下的dnsproxy.yml配置文件，取消#tls-port、#quic-port、#tls-crt、#tls-key前的注释，指定SSL证书位置，重启DNS服务即可
```bash
cd /usr/local/ednsplus/tools
./restart_dns
```

- 开启8053端口<br />
8053端口默认开启

- 开启防火墙端口<br />
请开启防火墙的UDP853、TCP853、UDP8053、TCP8053端口，部分有硬件防火墙服务的云主机应在管理页面同时开启以上端口

#### 如何使用DoT/DoQ/8053端口
只建议做被污染域名的上游DNS使用，hosts规则会失效

## 如何卸载服务
```bash
cd /usr/local/ednsplus/tools
./remove_service
```

## 源码鸣谢
[Overture](https://github.com/shawn1m/overture/releases/tag/v1.8)<br />
[Dnsproxy](https://github.com/AdguardTeam/dnsproxy/releases/tag/v0.43.1)<br />
[Dns2socks](https://github.com/rampageX/dns2socks)<br />
[Go-shadowsocks2](https://github.com/shadowsocks/go-shadowsocks2/releases/tag/v0.1.5)<br />

## Installation

For feedback, questions, and to follow the progress of the project: <br />
[Telegram Group](https://t.me/+VeV5wt1E6FA5Ue-x)<br />
