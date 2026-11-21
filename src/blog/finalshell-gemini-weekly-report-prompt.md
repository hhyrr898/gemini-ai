---
layout: article.njk
title: FinalShell 里放周报 Prompt，我用 Gemini 把要点变成稿子
description: 把我写周报的土办法搬到 FinalShell：先在本地整理要点，再用固定 Prompt 模板交给 Gemini 成稿。给出 3 步操作、模板结构和常见翻车点，适合每周要交汇报、又不想从零憋字的人。
date: 2026-11-21
generated: true
category: 工具教程
tags: ["移动端","会话管理","端口转发"]
heroImage: "/static/images/photo-1486406146926-c627a92ad1ab.jpg"
heroAlt: "FinalShell 里放周报 Prompt，我用 Gemini 把要点变成稿子 配图"
---

我在 FinalShell 4.5.12 的服务器目录里放了一个专门写周报的 txt，每到周五下午就把本周要点随手敲进去，再套用一段固定的 Prompt 抛给 Gemini。这个流程解决了最头疼的起笔问题。

这篇文章会完整拆解我的做法，看完你就能掌握一个可复用的半自动化工作流：如何整理要点、如何设计 Prompt、如何让 Gemini 生成一篇结构完整、语言通顺的周报初稿，省下反复琢磨措辞的时间。

这套流程不神奇，但它让我从「对着空白文档发呆」变成「往里填东西」。

## 三步落地

### 第一步：把要点写成清单

在服务器上新建 `~/notes/week.txt`，只写动词开头的短句。比如「修了登录超时」「联调支付回调」「评审下周排期」。别写形容词，别写句子。

这一步我在 FinalShell 里直接双击文件编辑，改完顺手存，比在聊天框里来回粘干净。想保留历史版本的话，可以参考[会话配置备份](/blog/finalshell-backup-session-config/)那个思路，给 notes 目录也做份备份。

### 第二步：套 Prompt 模板

模板我固定成四段，贴在文件末尾：

```
角色：你是我的周报代笔，语气平实。
输入：下面是本周要点，按项目分组。
输出：分「本周完成/进行中/风险与求助」三块，每条不超过 30 字。
限制：不要编造数据，缺信息就写「待补」。
```

要点和模板放同一个文件，复制一次就够，不用来回切窗口。

### 第三步：粘进 Gemini，只改不改写

把整段贴给 Gemini，出稿后我只做两件事：删掉它加的客套话，把「待补」的位置填上真实数字。命令历史里翻上周的要点也方便，[命令历史小技巧](/blog/finalshell-command-history-tips/)里提过按时间翻记录，我常用来找回上周写过的条目。

![FinalShell 编辑周报要点文件的界面](/static/images/photo-1486406146926-c627a92ad1ab.jpg)

## 几个我踩过的坑

第一版模板我写得太松，Gemini 给我补了一堆没发生的进度。后来加了「缺信息就写待补」，老实多了。

还有中文标点偶尔乱码，跟编码设置有关，[编码问题处理](/blog/finalshell-encoding-charset-fix/)里那套改法我试过，能救回来。

## 常见问题

**问：模板必须放服务器上吗？**

不必。放本地也行，我只是习惯和要点放一起，省得复制两次。

**问：Gemini 出的稿能直接用吗？**

不能。它负责结构，数字和结论得你自己核对，尤其是风险和求助那块。
## 延伸阅读

若需进一步查阅，可先看本站以下教程：

- [多环境会话树](/blog/finalshell-session-group-management/)
- [Linux 批量接入](/blog/finalshell-linux-ssh-setup/)
- [SSH 密钥轮换](/blog/finalshell-ssh-key-configuration/)
