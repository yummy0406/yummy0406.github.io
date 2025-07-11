---
title: Ubuntu2204版本国内git clone失败解决办法
date: 2025-06-17 16:49:00 +0800
categories: [Git]
tags: [Git]     
---

# 问题情况

国内访问github网站或者git clone时，时不时会出现无响应。换成镜像站方法不稳定，这里采用更改hosts文件解决。

```bash
ubuntu@virtual-machine:~/Desktop$ git clone https://github.com/XXX/xxx.git
正克隆到 'xxx'...
fatal: 无法访问 'https://github.com/XXX/xxx.git/'：GnuTLS recv error (-54): Error in the pull function.
```

多尝试几次后会出现443报错

```bash
ubuntu@virtual-machine:~/Desktop$ git clone https://github.com/XXX/xxx.git
正克隆到 'xxx'...
fatal: 无法访问 'https://github.com/XXX/xxx.git/'：Failed to connect to github.com port 443 after 21050 ms: 连接被拒绝
```

# 解决办法

## 打开hosts文件

```bash
sudo vim /etc/hosts
```
## 在hosts文件中加入以下内容

```vim
20.205.243.166                github.com
140.82.114.25                 alive.github.com
140.82.112.25                 live.github.com
185.199.108.154               github.githubassets.com
140.82.112.22                 central.github.com
185.199.108.133               desktop.githubusercontent.com
185.199.108.153               assets-cdn.github.com
185.199.108.133               camo.githubusercontent.com
185.199.108.133               github.map.fastly.net
199.232.69.194                github.global.ssl.fastly.net
140.82.112.4                  gist.github.com
185.199.108.153               github.io
140.82.114.4                  github.com
192.0.66.2                    github.blog
140.82.112.6                  api.github.com
185.199.108.133               raw.githubusercontent.com
185.199.108.133               user-images.githubusercontent.com
185.199.108.133               favicons.githubusercontent.com
185.199.108.133               avatars5.githubusercontent.com
185.199.108.133               avatars4.githubusercontent.com
185.199.108.133               avatars3.githubusercontent.com
185.199.108.133               avatars2.githubusercontent.com
185.199.108.133               avatars1.githubusercontent.com
185.199.108.133               avatars0.githubusercontent.com
185.199.108.133               avatars.githubusercontent.com
140.82.112.10                 codeload.github.com
52.217.223.17                 github-cloud.s3.amazonaws.com
52.217.199.41                 github-com.s3.amazonaws.com
52.217.93.164                 github-production-release-asset-2e65be.s3.amazonaws.com
52.217.174.129                github-production-user-asset-6210df.s3.amazonaws.com
52.217.129.153                github-production-repository-file-5c1aeb.s3.amazonaws.com
185.199.108.153               githubstatus.com
64.71.144.202                 github.community
23.100.27.125                 github.dev
185.199.108.133               media.githubusercontent.com
```
## 重启netrwork-manager

```bash
sudo service network-manager restart
```

