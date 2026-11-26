---
layout: article.njk
title: FinalShell 里用 Gemini 多模态看图写文案：我摸索出的操作流程
description: 分享在 FinalShell 中调用 Gemini 多模态能力看图写文案的实操步骤，包含环境准备、文件上传、提示词编写和结果查看，还解决编码报错、连接超时等常见问题。
date: 2026-11-26
generated: true
category: 技术教程
tags: ["会话管理","密钥登录","macOS客户端"]
heroImage: "/static/images/photo-1486406146926-c627a92ad1ab.jpg"
heroAlt: "FinalShell 里用 Gemini 多模态看图写文案：我摸索出的操作流程 配图"
---

2023 年 12 月 Gemini Pro Vision 开放 API 后，我琢磨着用它直接在 FinalShell 里批量处理产品图。这篇文章要解决的，正是在没有图形界面的服务器上，如何通过命令行调用 Gemini 的多模态能力，实现看图写文案的自动化流程。

整个过程并不一帆风顺，但最终我跑通了整个流程。下面我将分享完整的操作步骤和遇到的几个关键问题，帮你省去摸索的时间，直接上手实践。

## 第一步：确认你的 FinalShell 版本和服务器环境

我用的 FinalShell 是 4.5.1 版本，Windows 11 系统。先连上你的 Linux 服务器，在终端里敲 `python3 --version`，确保是 3.8 以上。Gemini 的 Python SDK 要求不算高，但太老的系统会缺依赖。

如果你还没配好 SSH 连接，可以参考这篇 [FinalShell SSH 连接教程](/blog/finalshell-ssh-connection-tutorial/)。连上之后，在 FinalShell 的左侧会话树里右键点击你的服务器，选“新建终端”，这样方便多窗口操作。

![FinalShell 左侧会话树界面](/static/images/photo-1486406146926-c627a92ad1ab.jpg)

然后安装必要的库：

```bash
pip install google-generativeai pillow
```

Pillow 用来处理图片，比如调整尺寸或转换格式，因为 Gemini 对图片大小有限制。

## 第二步：获取 API Key 并设置环境变量

去 Google AI Studio 申请一个 API Key，过程很简单，这里不展开。拿到之后，在服务器上设置环境变量：

```bash
export GOOGLE_API_KEY='你的密钥'
```

为了不用每次登录都设，可以写进 `~/.bashrc`。注意别把密钥提交到 Git 仓库。

## 第三步：上传图片并编写 Python 脚本

FinalShell 有个很方便的功能：直接拖拽文件到终端窗口就能上传。我把要写文案的产品图拖到 `/home/user/images/` 目录下。

然后新建一个 `gemini_copy.py` 文件，内容大概这样：

```python
import google.generativeai as genai
import os
from PIL import Image

genai.configure(api_key=os.environ['GOOGLE_API_KEY'])
model = genai.GenerativeModel('gemini-pro-vision')

img = Image.open('/home/user/images/product.jpg')
response = model.generate_content(['写一段电商文案，突出产品卖点，语气活泼', img])
print(response.text)
```

注意，Gemini Pro Vision 的模型名称是 `gemini-pro-vision`，别写成 `gemini-pro`，那个不支持图片。

## 第四步：运行脚本并查看结果

直接 `python3 gemini_copy.py` 运行。第一次可能会报错，说找不到模块，那是没装对。还可能会遇到编码问题，尤其是中文输出乱码。如果你也遇到，可以看看这篇 [FinalShell 编码问题解决](/blog/finalshell-encoding-charset-fix/)，里面提到了修改终端编码的方法。

运行成功后，终端会打印出文案。我试了一张蓝牙耳机的图，它写出来还挺像那么回事。

## 常见问题

**1. 报错 `ImportError: cannot import name 'genai'`**

多半是没安装 `google-generativeai` 或者版本不对。用 `pip show google-generativeai` 检查一下，如果没有就重装。另外注意包名是 `google-generativeai`，不是 `google-generative-ai`。

**2. 上传图片后脚本找不到文件**

FinalShell 拖拽上传默认会放到当前终端的工作目录，但你脚本里写的路径可能是绝对路径。用 `pwd` 确认一下当前目录，或者改用相对路径。

**3. 返回内容为空或报安全错误**

Gemini 对图片内容有安全过滤，如果图片里有敏感元素，它可能拒绝生成。换张图试试，或者调整提示词，明确说明用途。

**4. 连接超时**

如果你的服务器在国内，访问 Google API 可能不稳定。可以设置代理，或者把脚本放到能访问的机器上跑。我在 FinalShell 里用 `export https_proxy=...` 临时解决。

## 一些提醒

我踩过一个坑：一开始用 `gemini-pro` 模型传图片，总是报错，后来才发现模型选错了。另外，图片别太大，超过 4MB 会被拒绝，最好先用 Pillow 压缩一下。

如果你经常需要切换会话，记得把常用配置备份，可以参考 [FinalShell 备份会话配置](/blog/finalshell-backup-session-config/) 这篇。命令行历史也很有用，按上箭头就能找回之前的命令，省得重敲。

这套流程跑通之后，批量写文案就快多了，虽然偶尔要人工润色，但至少不用从零开始。
## 延伸阅读

若需进一步查阅，可先看本站以下教程：

- [Linux 批量接入](/blog/finalshell-linux-ssh-setup/)
- [SSH 密钥轮换](/blog/finalshell-ssh-key-configuration/)
- [Windows 生产部署](/blog/finalshell-windows-install-guide/)
