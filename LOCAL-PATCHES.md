# 本地补丁清单（本 fork 私有）

这仓库是 `NousResearch/hermes-agent` 的 fork。下面记录**本机自己的改动**。
每次从上游同步后，按文末步骤检查补丁还在不在。

> 当前本地补丁一共 **1 个**。

---

## 补丁 1：opencode-go 卡死处理（stale stream timeout）

- **文件**：`agent/chat_completion_helpers.py`
- **commit**：`local-stale-tuning`（tag）
- **配套配置**：`~/.hermes/config.yaml` → `providers.opencode-go.stale_timeout_seconds: 60`
  （配置文件在仓库外，`hermes update` 不会动它）

### 改了什么

1. `_StreamingCall._resolve_stale_timeout`
   显式配置的 `stale_timeout_seconds`（或环境变量 `HERMES_STREAM_STALE_TIMEOUT`）**直接生效**，
   不再被 `max(base, 推理模型下限)` 和上下文放大盖住。→ 大上下文会话能压到 60s。

2. `_check_stale_giveup`
   连续卡死的放弃阈值默认 **5 → 30**。→ 中转瞬时抖动不会再把长任务整条干掉。

### 为什么

opencode-go 中转在**长响应 / 大上下文**（实测 22 万–33 万 token）会随机卡死：
一个 chunk 都不回，或直接 RST / `HTTP2 framing error`。

- 旧行为：卡死要等 600s 才判定；且连续 5 次就永久放弃整条调用。
- 现在：60s 就判定并自动重试；连续 30 次（约 30 分钟）后才放弃。

### 验证补丁还在

```bash
cd ~/.hermes/hermes-agent
grep -n 'HERMES_STREAM_STALE_GIVEUP", 30' agent/chat_completion_helpers.py   # 应有一行
grep -n 'if explicit:' agent/chat_completion_helpers.py                      # 应有一行
```

### 调整

| 想改什么 | 怎么改 |
|---|---|
| 卡死判定时间 | `~/.hermes/config.yaml` → `providers.opencode-go.stale_timeout_seconds` |
| 放弃阈值 | 环境变量 `HERMES_STREAM_STALE_GIVEUP`（`0` = 永不放弃） |

改完都要**重启 Hermes** 才生效。

---

## 从上游同步之后要做什么

> ⚠️ main 上有本地 commit，所以 **`git pull --ff-only upstream main` 会失败**（无法快进）。
> 用下面的方式：

```bash
cd ~/.hermes/hermes-agent

# 1) 拉取
git fetch upstream
git fetch origin

# 2) 合并上游（保留本地补丁）
git merge upstream/main        # 只动两个函数，一般不会冲突；冲突就手动解决

# 3) 检查补丁是否被上游覆盖
grep -n 'HERMES_STREAM_STALE_GIVEUP", 30' agent/chat_completion_helpers.py
grep -n 'if explicit:' agent/chat_completion_helpers.py

# 4) 若补丁没了，重新打上
git cherry-pick local-stale-tuning

# 5) 推回自己的 fork
git push origin main
git push origin local-stale-tuning
```

## 补丁丢了怎么找回

tag 在 fork 上，任何时候都能捞回来（重装、`reset --hard` 之后也行）：

```bash
git fetch origin
git cherry-pick local-stale-tuning
```

列出所有本地补丁（相对上游多出来的 commit）：

```bash
git log --oneline upstream/main..main
```

## 撤销某个补丁

```bash
git revert local-stale-tuning                                   # 保留历史地撤销
git checkout upstream/main -- agent/chat_completion_helpers.py  # 或直接还原成上游版本
```

## 注意

- **推之前别用 `git reset --hard` 回退** —— 那是唯一会真丢补丁的操作。
- `hermes update` 只从 `origin` 做快进合并，不受本地 commit 影响；
  但**前提是补丁已经 push 到 origin**。
