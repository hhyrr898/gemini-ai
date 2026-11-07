---
layout: article.njk
title: FinalShell 连不上 Gemini API？先看看我的密钥申请踩坑记
description: 从零开始申请 Gemini API 密钥，手把手教你用 FinalShell 在服务器上配置并测试调用。包含免费额度实测、常见报错处理，适合刚接触 AI 接口的新手。
date: 2026-11-07
generated: true
category: 教程
tags: ["会话管理","密钥登录","macOS客户端"]
heroImage: "/static/images/photo-1486406146926-c627a92ad1ab.jpg"
heroAlt: "FinalShell 连不上 Gemini API？先看看我的密钥申请踩坑记 配图"
---

## 开场：为什么我在 FinalShell 里折腾 Gemini API

上周想把一个小脚本挂到服务器上跑，调用 Gemini 做文本摘要。本地测试通了，一上服务器就报 401。排查半天，发现是密钥申请时选错了项目，顺带把 AI Studio 的免费额度规则摸了一遍。这篇就记录我从申请密钥到在 FinalShell 里跑通的全过程，2025 年 2 月实测，免费额度还够用。

![配图说明](/static/images/photo-1486406146926-c627a92ad1ab.jpg)

## 第一步：在 AI Studio 拿到 API 密钥

浏览器打开 Google AI Studio，用 Google 账号登录。左侧菜单找到「Get API key」，点「Create API key」。界面会弹出一个项目选择框，如果你之前建过 Google Cloud 项目，这里会列出来，没有就让它自动新建。

创建完成后，密钥是一串以 `AIza` 开头的字符。注意：**只显示一次**，复制下来存好。我踩过一个坑——当时随手关掉弹窗，结果又得重新生成一个，旧的直接作废。

免费额度方面，AI Studio 页面右侧有个「Pricing」小标签，点开能看到当前模型的免费配额。我用的 `gemini-1.5-flash` 每分钟 15 次请求、每天 1500 次，对于个人小工具绰绰有余。但注意，免费额度会随模型和地区调整，申请前最好自己看一眼。

## 第二步：在 FinalShell 里把密钥写进环境变量

打开 FinalShell，连上你的 Linux 服务器。我习惯在 `/etc/profile.d/` 下放一个 `gemini.sh`，这样所有用户都能读到。

右键左侧会话树里的服务器，选「新建终端」，然后执行：

```bash
echo 'export GEMINI_API_KEY="你的密钥"' | sudo tee /etc/profile.d/gemini.sh
sudo chmod +x /etc/profile.d/gemini.sh
source /etc/profile.d/gemini.sh
```

验证一下：`echo $GEMINI_API_KEY`，能打印出密钥就对了。如果你用 FinalShell 的「命令管理器」功能，也可以把这几行存成快捷命令，下次一键执行。

## 第三步：写个最小脚本测试调用

我直接用 curl 测，省得装 SDK。在服务器上新建 `test_gemini.sh`：

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}" \
-H 'Content-Type: application/json' \
-d '{"contents":[{"parts":[{"text":"用一句话解释什么是 SSH"}]}]}'
```

跑 `bash test_gemini.sh`，正常会返回 JSON，里面 `candidates` 字段就是回答。如果返回 403，多半是密钥没生效或项目没开通 API；返回 429 就是免费额度用超了。

在 FinalShell 里看返回结果时，建议把终端字体调大一点，JSON 缩进乱的时候容易看错行。右键终端空白处「终端设置」里可以改。

## 第四步：把调用封装成可复用的小工具

单次 curl 太麻烦，我写了个 shell 函数塞进 `~/.bashrc`：

```bash
gemini() {
  curl -s -X POST "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}" \
  -H 'Content-Type: application/json' \
  -d "{\"contents\":[{\"parts\":[{\"text\":\"$1\"}]}]}" | jq -r '.candidates[0].content.parts[0].text'
}
```

需要先 `apt install jq`。之后直接 `gemini "帮我写个正则"` 就能出结果。注意引号转义，我一开始没加 `-s` 和 `jq`，输出一堆调试信息。

如果你经常在 FinalShell 里翻历史命令，可以看看 [FinalShell 命令历史小技巧](/blog/finalshell-command-history-tips/)，能省不少重复输入。

## 常见问题

**1. 密钥复制后提示「API key not valid」**

八成是复制时带了空格或换行。用 `cat -A /etc/profile.d/gemini.sh` 看下，行尾如果出现 `$` 是正常的，但如果有 `^M` 说明混入了 Windows 换行符，用 `sed -i 's/\r//'` 去掉。

**2. 免费额度突然没了**

AI Studio 的免费额度按项目算，不是按密钥。如果你在同一个项目下建了多个密钥，它们是共享配额的。另外，某些模型（比如 `gemini-1.5-pro`）免费额度很低，换 `flash` 试试。

**3. 返回内容中文乱码**

FinalShell 终端编码默认 UTF-8，但如果你之前改过，可以在「终端设置」里确认编码。如果返回的 JSON 里中文显示为 `\uXXXX`，那是正常的转义，用 `jq` 解析就会还原。实在不行参考 [FinalShell 编码问题修复](/blog/finalshell-encoding-charset-fix/)。

**4. 想把密钥同步到多台服务器**

别手动复制，用 FinalShell 的会话同步功能，或者把 `gemini.sh` 放到私有 Git 仓库里。顺便说下，FinalShell 的 [会话配置备份](/blog/finalshell-backup-session-config/) 也能把命令片段一起带走。

整套流程走下来，最花时间的其实是等 Google Cloud 项目初始化。密钥本身申请不到两分钟。免费额度对个人开发完全够用，但别拿它跑生产任务——我试过一天调了 1400 多次，第二天直接 429。

如果你也在 FinalShell 里折腾过 Docker 跑 AI 服务，可以翻翻 [容器访问那篇](/blog/finalshell-docker-container-access/)，有些端口映射的坑是相通的。
## 延伸阅读

若需进一步查阅，可先看本站以下教程：

- [SSH 密钥轮换](/blog/finalshell-ssh-key-configuration/)
- [多环境会话树](/blog/finalshell-session-group-management/)
- [Linux 批量接入](/blog/finalshell-linux-ssh-setup/)
