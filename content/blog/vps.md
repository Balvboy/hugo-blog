---
title: "VPS搭建步骤"
date: 2021-11-29T10:42:21+08:00
tags:
    - vps
draft: true
---

# VPS搭建步骤

```shell
# 切换到root用户
sudo -i 

# 使用脚本安装v2ray
bash <(curl -s -L https://git.io/v2ray.sh)

# 按照提示配置v2ray

# 关闭系统防火墙
systemctl stop firewalld

# 查看防火墙状态
systemctl status firewalld

# 放开云服务器的端口限制，不同的云平台配置不同

```


