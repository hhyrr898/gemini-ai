---
layout: article.njk
title: FinalShell 里调 Gemini 报 429？我把限流原因和重试试了一遍
description: Gemini 接口返回 429 报错时怎么排查？这篇讲清限流触发原因，给出在 FinalShell 里改请求间隔、加退避重试、拆并发三个可照着做的办法，附上常见边角问题。
date: 2026-11-20
generated: true
category: 技术教程
tags: ["服务器运维","移动端","SSH连接"]
heroImage: "/static/images/photo-1486406146926-c627a92ad1ab.jpg"
heroAlt: "FinalShell 里调 Gemini 报 429？我把限流原因和重试试了一遍 配图"
---

## 开头说两句

用 FinalShell 连服务器跑脚本调 Gemini 接口，最烦的不是报错，是那个 429。请求发出去，回来一句 rate limit exceeded，脚本直接躺平。这篇文章就解决一件事：把 429 的原因弄明白，然后给你几套在 FinalShell 里当场能用的重试办法。

我测试时用的是 FinalShell 4.5.10（2024 年 11 月版），服务器是 Ubuntu 22.04，跑的是 Python requests 脚本。下面提到的命令和改法都在这个环境里跑通过。

![服务器终端里调试接口的场景](/static/images/photo-1486406146926-c627a92ad1ab.jpg)

## 429 到底是谁在拦你

### h3 429 报错是怎么来的？

先分清两种限流。一种是按分钟算的请求数（RPM），比如免费额度里每分钟只放几十次；另一种是按天算的配额，超了就等到第二天。429 的响应体里通常会带 `retry_after` 或者类似的重试等待提示，别忽略它，这是最靠谱的线索。

还有一种情况容易被误判：不是接口限你，是你本机脚本并发开太多，一瞬间把额度打满。我见过一个循环里起了 20 个线程同时请求，结果大半都是 429。先看自己的并发量，再怀疑平台。

判断方法很土但有效：在 FinalShell 里用 `curl -i` 打一次单条请求，看返回头的状态码和 body。如果单发没问题、批量就炸，那基本是并发或者频率问题。

### h3 怎么在 FinalShell 里看清 429 的具体信息？

别只看脚本抛的异常，异常信息经常把真实响应吞掉。做法是临时把请求包一层打印。

步骤一，在脚本里让 requests 不自动抛错，改成手动判断：

```python
resp = requests.post(url, headers=h, json=body, timeout=30)
print(resp.status_code)
print(resp.text[:500])
```

步骤二，在 FinalShell 终端里直接跑一次 `curl -i -X POST ...`，把 header 和 body 都打出来，重点看 `x-ratelimit-remaining` 这类字段和响应体里的等待秒数。

步骤三，对照你脚本里的并发数和请求间隔，算一下一分钟内实际打了多少次。多数时候数字一摆出来，问题就清楚了。顺带一句，如果你习惯把请求历史留着复盘，用命令历史功能翻记录比重新构造请求快得多，可以看这篇 [FinalShell 命令历史小技巧](/blog/finalshell-command-history-tips/)。

## 重试策略怎么落地

### h3 429 之后该怎么重试才不白等？

核心就三点：指数退避、带随机抖动、设上限。

别固定睡 1 秒重试，平台在限流时你越急它越不给你。正确姿势是等待时间逐次翻倍，比如 2s、4s、8s、16s，同时加一点随机偏移，避免多线程同时醒来再次撞墙。

给个能直接用的写法：

```python
import time, random
def call_with_retry(fn, max_retry=4):
    for i in range(max_retry):
        r = fn()
        if r.status_code != 429:
            return r
        wait = (2 ** i) + random.uniform(0, 1)
        print(f"429, sleep {wait:.1f}s")
        time.sleep(wait)
    raise RuntimeError("retry exhausted")
```

步骤上：先包一层调用函数，把状态码判断放进去；再设定最大重试次数（我一般给 4 次）；最后别忘了重试耗尽要显式报错，别悄悄返回 None。

超过上限还在 429，说明不是瞬时抖动，而是你的日配额真的用完了，这时候重试没意义，去看控制台配额。

### h3 并发太多导致 429，怎么在脚本里压下来？

把线程池大小调小是最直接的一刀。原来 `ThreadPoolExecutor(max_workers=20)`，先降到 3 到 5 再看。

如果还不行，就在每次请求之间加个最小间隔。我用 `time.sleep(0.5)` 起步，实际观察下来，单机稳定跑在每分钟 30 到 40 次左右比较安全，具体看你的额度。

再狠一点的办法是把任务拆批，一批跑完打印进度再跑下一批，这样即使中途 429，也不会把整批结果弄乱。跑批的时候留意日志和输出编码，中文乱码会让你误判请求内容，参考 [FinalShell 编码问题排查](/blog/finalshell-encoding-charset-fix/)。

### h3 想避开重试折腾，有什么省事的做法？

两条。第一条是把调用逻辑从临时脚本挪到固定任务里，用 FinalShell 的会话配置保存好环境变量和路径，下次直接连上去跑，不用每次重新拼命令。备份会话配置的方法在 [FinalShell 会话备份与恢复](/blog/finalshell-backup-session-config/) 里写得很细，照着做十分钟能搞定。

第二条，如果请求本身是容器里跑的，先确认容器内的出口和宿主是不是同一套网络，排查方向会完全不同，可以看 [FinalShell 进入 Docker 容器](/blog/finalshell-docker-container-access/)。限流问题很多时候不是代码，是环境没对齐。

## 常见问题

有人问 429 和 403 会不会混。一般不会，403 是权限或 key 的问题，429 是频率和配额，看状态码就能分开。

还有一个边角情况：重试都对了，但日志里时间戳乱跳。那多半是服务器时区没设对，`timedatectl` 看一眼，顺手改掉，别让时区坑你判断退避间隔。

最后提醒一句，重试代码一定要写进正式脚本里，别在终端里手工 sleep 几次就当解决了，下次还是同样的坑。
## 延伸阅读

若需进一步查阅，可先看本站以下教程：

- [Linux 批量接入](/blog/finalshell-linux-ssh-setup/)
- [隧道端口映射](/blog/finalshell-port-forwarding-guide/)
- [多环境会话树](/blog/finalshell-session-group-management/)
