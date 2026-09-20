# 本地补丁清单（本 fork 私有）

这仓库是 `NousResearch/hermes-agent` 的 fork。下面记录**本机自己的改动**。
每次从上游同步后，按文末步骤检查补丁还在不在。

> 当前本地补丁：**1 个**（外加 1 个配置项，配置在仓库外、不在 git 里）。
> 最近一次上游同步：2026-09-20（合并了 5152 个上游 commit）。

---

## 补丁 1：放宽 stale 熔断阈值（opencode-go）

- **文件**：`agent/chat_completion_helpers.py` → `_check_stale_giveup()`
- **commit / tag**：`local-stale-tuning`
- **改动**：连续卡死后的放弃阈值，默认 **5 → 30**

### 为什么要改

opencode-go 中转在**长响应 / 大上下文**（实测 22 万–33 万 token）会随机卡死：
一个 chunk 都不回，或直接 RST / `HTTP2 framing error`。

旧的 5 次阈值下，一次**瞬时**抖动（持续几分钟）就能把跑了很久的长任务整条干掉
（报错：`Provider has been unresponsive ... for N consecutive stale attempts`）。

30 次 ≈ 60s 阈值下"连续卡死 30 分钟"才放弃：能扛过瞬时抖动，又不会在中转真挂时无限重试。

### 怎么调

环境变量 `HERMES_STREAM_STALE_GIVEUP`：`0` = 永不放弃；正数 = 自定义次数。

---

## 配置项（不在 git 里，`hermes update` 不会动它）

`~/.hermes/config.yaml`：

```yaml
providers:
  opencode-go:
    stale_timeout_seconds: 60
```

作用：把 opencode-go 的"卡死判定时间"从默认 180s（大上下文还会被放大到 300s，
且被推理模型下限抬到 600s）压到 **60s**。

> ✅ **这一条上游已原生支持**（新增 `_cloud_stale_timeout_for()`，#115024）：
> 显式配置的 `stale_timeout_seconds` 会直接生效，不再被任何下限覆盖。
> 所以**本地不需要为此打补丁**，只留配置即可。

### 验证配置 + 补丁都生效

```bash
cd ~/.hermes/hermes-agent
./venv/bin/python -c "
from types import SimpleNamespace
from agent.chat_completion_helpers import _StreamingCall, _check_stale_giveup
a=SimpleNamespace(provider='opencode-go',model='deepseek-v4.1-flash',base_url='https://opencode.ai/zen/go/v1')
s=SimpleNamespace(agent=a,api_kwargs={'model':'deepseek-v4.1-flash'})
_StreamingCall._resolve_stale_timeout(s)
print('stale threshold =', s._stream_stale_timeout, 's   (期望 60.0)')
for n in (5,29,30):
    try: _check_stale_giveup(SimpleNamespace(_consecutive_stale_streams=n)); r='retries'
    except RuntimeError: r='ABORTS'
    print(f'  streak={n} -> {r}   (期望 5/29 retries, 30 ABORTS)')
"
```

---

## 从上游同步之后要做什么

> ⚠️ main 上有本地 commit，所以 **`git pull --ff-only upstream main` 一定会失败**（无法快进）。
> 改用 `git merge`：

```bash
cd ~/.hermes/hermes-agent

# 1) 拉取
git fetch upstream && git fetch origin

# 2) 合并上游（冲突大概率只出现在 agent/chat_completion_helpers.py）
git merge upstream/main

# 3) 检查补丁还在不在（应输出一行 30）
grep -n 'HERMES_STREAM_STALE_GIVEUP", 30' agent/chat_completion_helpers.py

# 4) 若被上游覆盖 / 冲突时丢了，重新打上
git cherry-pick local-stale-tuning

# 5) 依赖可能变了（pyproject.toml / uv.lock），保险做法跑一次 update
hermes update            # 会重装依赖、迁移配置、重建桌面端、重启服务

# 6) 推回自己的 fork
git push origin main
```

### 万一冲突

`agent/chat_completion_helpers.py` 是唯一会冲突的文件。**优先保留上游**，再把补丁重打一遍：

```bash
git checkout --theirs agent/chat_completion_helpers.py   # 取上游版本
# 然后手动把 _check_stale_giveup() 里的默认值 5 改成 30（同补丁 1）
```

---

## 补丁丢了怎么找回

tag 在 fork 上，任何时候都能捞：

```bash
git fetch origin
git cherry-pick local-stale-tuning
```

列出所有本地补丁（相对上游多出来的 commit）：

```bash
git log --oneline upstream/main..main
```

## 撤销

```bash
git revert local-stale-tuning      # 保留历史地撤销
# 或丢弃全部本地改动，回到纯上游：
git reset --hard upstream/main
```

## 注意

- **推之前别 `git reset --hard` 回退** —— 那是唯一会真丢补丁的操作。
- `hermes update` 只从 `origin` 快进合并，不受本地 commit 影响；但**前提是补丁已 push 到 origin**。
- 代码改动都要**重启 Hermes** 才生效（配置项也同理）。
