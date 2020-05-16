---
title: windows下github访问速度慢的解决方法
date: 2020-01-04 17:19:59
tags: github
---



解决windows下github访问速度慢的问题，方法参考了[路人Q的博客](<https://www.cnblogs.com/lurenq/archive/2019/12/10/12014415.html>)。该方法也同样适用于所有其他的网站。



### 1、修改hosts文件

以管理员的身份打开hosts文件（C:\Windows\System32\drivers\etc\hosts），并添加如下内容：

```shell
151.101.44.249 github.global.ssl.fastly.net
192.30.253.112 github.com
103.245.222.133 assets-cdn.github.com
23.235.47.133 assets-cdn.github.com
203.208.39.104 assets-cdn.github.com
204.232.175.78 documentcloud.github.com
204.232.175.94 gist.github.com
107.21.116.220 help.github.com
207.97.227.252 nodeload.github.com
199.27.76.130 raw.github.com
107.22.3.110 status.github.com
204.232.175.78 training.github.com
207.97.227.243 www.github.com
185.31.16.184 github.global.ssl.fastly.net
185.31.18.133 avatars0.githubusercontent.com
185.31.19.133 avatars1.githubusercontent.com
192.30.253.120 codeload.github.com 
```



> 注意：如果以上域名的ip地址发生变化则需要重新根据第3章节中的方法重新查询新ip然后重新更新dns缓存



### 2、更新dns缓存



打开cmd命令行工具，并执行以下命令来强制更新dns

```shell
ipconfig /flushdns
```



### 3、添加其他网站



如果还需要增加其他网站的dns缓存则直接到[ipaddress网站](https://www.ipaddress.com)上查询对应的ip即可，然后在根据第1、2章节重新修改hosts文件并更新dns缓存即可。