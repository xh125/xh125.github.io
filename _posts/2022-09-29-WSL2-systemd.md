---
layout:     post
title:      "WSL2中引入systemd"
date:       2022-09-29 12:56:00
author:     "Xiehua"
header-img: "images/post/post-bg-understand.jpg"
tags:
    - WSL
    - systemd
---
## WSL2中引入systemd

**参考来源:**[开源中国OSC][2]
微软和 Canonical 联合宣布，systemd 现在可以在 Windows Subsystem for Linux（WSL2）中运行了，此举可以让用户在 Windows 设备上获得更加全面的 Linux 体验。systemd 的作者 Lennart Poettering 在 7 月份离开红帽并加入了微软。

systemd 是一套用于 Linux 系统的基本构建模块，它提供了一个系统和服务管理器，作为 PID 1 运行并启动系统的其他部分。
许多知名的 Linux 发行版（如 Ubuntu、Debian 等）都默认运行 systemd，这一变化意味着 WSL 允许你使用依赖于 systemd 支持的软件，也让 WSL 更贴近于那种在设备上独立安装运行的 Linux 发行版而不是兼容层。
依赖 systemd 的一些知名 Linux 应用程序包括：

- snap（Canonical 为使用 Linux 内核和 systemd init 系统的操作系统开发的软件打包和部署系统）
- microk8s（一个轻量级的 Kubernetes，旨在降低 K8s 和云原生应用开发的准入门槛）
- systemctl（检查和控制 systemd 系统和服务管理器的状态）

### 如何在 Ubuntu WSL 中启用 systemd
• 要使用`systemd`，首先需确保运行的是来自 Microsoft Store 且版本号为 0.67.6 及以上版本的 WSL，用户可以运行 `wsl --version` 来检查版本号。
如果你正在运行旧版本，你可以通过 微软应用商店(Microsoft Store) 或者以下命令更新它。

```bash
wsl --update
```

• 其次需要在 Ubuntu 实例中，将以下修改内容添加到 /etc/wsl.conf 中：
```bash
[boot]
systemd=true
```
• 然后通过在 PowerShell 中运行 `wsl --shutdown` 来重启实例，并重新启动 Ubuntu


**Note:** 文章中可能会有问题，欢迎指正，如有建议或者遇到问题，欢迎在评论区留言。评论系统采用了[Disqus系统][1]，需要翻墙才能加载。

[1]:https://disqus.com/
[2]:https://www.toutiao.com/article/7146405826788360717/?app=news_article&timestamp=1664421421&use_new_style=1&req_id=2022092911170001013304216609222DDB&group_id=7146405826788360717&wxshare_count=1&tt_from=weixin&utm_source=weixin&utm_medium=toutiao_android&utm_campaign=client_share&share_token=42e6e2e3-c613-4ae6-8373-393cdfb786ed&source=m_redirect
