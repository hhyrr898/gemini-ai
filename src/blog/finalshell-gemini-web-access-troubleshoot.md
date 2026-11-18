---
layout: article.njk
title: FinalShell 里排查 Gemini 网页打不开，我花了 20 分钟
description: 2026-06-20 我遇到 Gemini 网页版打不开，用 FinalShell 4.5.12 的 SSH 隧道和 curl 测试查出是地区限制和 DNS 污染。这篇记录三步排查法，帮你快速定位是网络问题还是浏览器问题。
date: 2026-11-18
generated: true
category: 技术教程
tags: ["移动端","会话管理","端口转发"]
heroImage: "/static/images/photo-1486406146926-c627a92ad1ab.jpg"
heroAlt: "FinalShell 里排查 Gemini 网页打不开，我花了 20 分钟 配图"
---

2026-06-20 下午，Gemini 网页版报错无法访问，我打开 FinalShell 4.5.12 开始排查。这篇文章解决特定网站打不开但其他网页正常的网络排查难题，看完后，你能在仅有 SSH 终端可用时，掌握用命令行层层拆解 DNS 解析与连通性，从而快速定位此类故障。

当时其他网站均

## 第一步：用 curl 确认是 DNS 还是连接被拒

在 FinalShell 里连上 VPS，执行：

```
curl -I --max-time 10 https://gemini.google.com
```

返回 `Could not resolve host`。这说明不是浏览器问题，是域名解析失败。我换用 `nslookup gemini.google.com 8.8.8.8` 对比，本地 DNS 返回了奇怪的地址，而公共 DNS 正常。基本确定是 DNS 污染导致的地区限制表现。

如果你也遇到类似情况，可以先参考 [FinalShell 编码问题排查](/blog/finalshell-encoding-charset-fix/) 里的思路，先分清是工具问题还是网络问题。

## 第二步：在 FinalShell 里开 SSH 隧道做对照测试

我一般会先用隧道把浏览器流量转到 VPS 上，看能不能打开。FinalShell 的隧道功能在连接右键菜单里，填本地端口 1080，类型选 SOCKS5。

设置好后，浏览器配代理 `127.0.0.1:1080`，再次访问 Gemini。这次页面能加载出来，说明本地网络到 Google 的路径被阻断了，VPS 的出口是通的。

这一步能快速区分是「网站挂了」还是「你这边网络到不了」。顺手把这次会话配置备份一下，下次不用重填，方法在 [FinalShell 备份会话配置](/blog/finalshell-backup-session-config/) 里写过。

## 第三步：换 DNS 并清理浏览器缓存

既然隧道能通，我就把本机 DNS 改成 `1.1.1.1` 和 `8.8.8.8`，再执行 `ipconfig /flushdns`。浏览器里清掉 Gemini 的站点数据，重新打开。这次直连也成功了。

![配图说明](/static/images/photo-1486406146926-c627a92ad1ab.jpg)

如果直连还是不行，就保持隧道方案，至少能正常用。另外，如果你平时用 FinalShell 连容器调试，可以看看 [Docker 容器访问](/blog/finalshell-docker-container-access/) 里的端口映射思路，排查网络时挺有用。

## 常见问题

**Q：为什么手机热点也打不开？**
A：地区限制通常按 IP 段判断，热点和宽带可能走同一出口，所以表现一样。换 DNS 或走隧道才能绕开。

**Q：FinalShell 隧道一直连不上怎么办？**
A：先确认 VPS 的 SSH 端口通不通，再检查本地端口有没有被占用。我遇到过 1080 被其他软件占用，换 1081 就好了。
## 延伸阅读

若需进一步查阅，可先看本站以下教程：

- [隧道端口映射](/blog/finalshell-port-forwarding-guide/)
- [Linux 批量接入](/blog/finalshell-linux-ssh-setup/)
- [多环境会话树](/blog/finalshell-session-group-management/)
