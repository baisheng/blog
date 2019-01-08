---
title: 我用 WSL 做些什么
date: 2018-12-05 09:39:00
tags:
---

## nginx

平时写一些前端项目的时候, 为了处理调试时的跨域问题, 常常使用 nginx 来配置一下反向代理, 可以又偏偏碰上`nginx for win`非常不好用

这个时候如果直接在 WSL 里面配置安装 nginx, 就能免去在 windows 环境下配置的痛苦

配置方式和 linux 主机没有两样

```bash
apt install nginx
service nginx start
cd /etc/nginx/sites-enabled
vi xxx.conf
service nginx reload
```

## 一些坑

### IO 性能

