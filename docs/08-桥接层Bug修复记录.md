# 08 · 桥接层 Bug 修复记录

> 三个真实 bug，症状都是「机器人不回话」。

## Bug 1 · 暂存的 @ 被误丢弃

### 症状

群里 @ 了机器人，它不回。日志里只有：

```
会话繁忙，暂存唤醒原因 group:1049197747（atMention@seq165）
繁忙期间唤醒 atMention@seq165 已被当前回合处理，跳过补发
```

### 根因

`src/bridge.js` 里判断暂存的唤醒是否还有效：

```js
// 原判据
const stillRelevant = stEnd.unread.some(m => Number(m.seq) >= Number(item.seq));
```

**但 AI 调 `qq_set_wake_config(throughSeq)` 时会顺带把未读标记为已读** ——
于是所有待补发的 @ 都被判定「已处理」而丢弃，**即使它一条都没回**。

### 修复

判据从「读过没有」改成「回过没有」：

```js
// [FIX] 原判据用 unread 判断「是否已被处理」，但 qq_set_wake_config(throughSeq)
// 会顺带清空 unread —— 于是所有待补发的 @ 被误判为「已处理」而丢弃。
// 改成看「那条消息之后机器人有没有真的发过话」：读过 ≠ 回过。
const rmEnd = Array.isArray(stEnd.recentMessages) ? stEnd.recentMessages : [];
const idxAt = rmEnd.findIndex(m => m && !m.isSelf && Number(m.seq) >= Number(item.seq));
const repliedAfter = idxAt >= 0 ? rmEnd.slice(idxAt).some(m => m && m.isSelf) : false;
const stillRelevant = !repliedAfter;
```

## Bug 2 · 回合无限长导致会话永久繁忙

### 症状

```
未读=26（持续增长）
last turn/start: turn 8   564 秒前
last turn/end:   turn 7   766 秒前
⚠️ start 之后没有 end
```

期间所有 @ 全部排队，机器人在群里「装死」。

### 根因

群聊高峰期 `qq_wait_for_messages` 一直返回 `arrived: true`，
模型就在**同一个 turn 里**「发一条 → 等消息 → 再发」，turn 永不结束。

而 `isConversationBusyV2()` 的判据是：

```js
if (sid && (v2TurnStartAt.has(sid) || collectors.has(sid))) return true;
```

**只要 turn 没结束就一直忙**，而 turn 没有时长上限。

### 修复

在 `/api/socialV2/wait` 加回合时长保护 —— 超过上限就不再等，逼它收尾：

```js
// [TURNGUARD] 回合时长保护
const turnSid = state.sessions[key];
const turnStartAt = turnSid ? (v2TurnStartAt.get(turnSid) ?? 0) : 0;
const maxTurnMs = Number(cfg.socialV2?.wait?.maxTurnMs) || 240000;
if (turnStartAt && Date.now() - turnStartAt > maxTurnMs) {
  sendJson({
    ok: true, arrived: false, quiet: true, timeout: true, waitedMs: 0,
    mustEnd: true,
    hint: '本回合已经跑太久。不要再等消息了：现在调用 qq_set_wake_config 结束本回合。' +
          '还没回的消息不会丢，会作为新的唤醒再进来。',
    newMessages: [], readThroughSeq: readThroughSeqV2(stg), unreadCount: stg.unread.length
  });
  return;
}
```

### 踩过的坑

`maxTurnMs` 一开始设 100 秒，但 `preSleepWaitMs`（沉睡前观察）是 300 秒 ——
**回合被掐断，观察永远做不完，`set_wake_config` 一直失败**，
唤醒配置反复被重置为默认。

模型的思考里连说三次：

> *本回合到了时间上限，收尾时又被观察窗口挡了一下，没能设置成功。*

**修法：观察窗口必须小于回合上限。**

```
preSleepWaitMs:  300000 → 75000   （5 分钟 → 75 秒）
maxTurnMs:       100000 → 200000  （100 秒 → 200 秒）
```

## Bug 3 · dsh 内存抖动，进程卡进 D 状态

### 症状

```
PID 572910  dsh      486MB  Dsl   ← D 状态（uninterruptible sleep = 等磁盘 IO）
dsh HTTP:  10 秒超时 → 9.4 秒 → 0.76 秒
dshReady=false
Swap 用了 1641 MB，zram 1G 已满
```

### 根因

2G 内存的机器跑着 dsh(486MB) + snowluma 容器(268MB) + 桥接(48MB) + 系统(300MB)。
跑几天后 swap 挤爆，页面被换到磁盘，每次访问都等 IO。

### 修复

`/usr/local/bin/memguard` + systemd timer（每 5 分钟）：

```bash
THRESHOLD_MB=280       # 可用内存低于此值 → 重启 dsh
# swap 使用率 >= 60% → 重启 dsh
# 冷却期 30 分钟，避免连续重启
```

**阈值定在 280MB / 60% 的原因**：实测炸掉的时候是可用 376MB / swap 53%，
所以要在那之前动手。

重启后：

```
dsh HTTP  9.4 秒 → 0.002 秒
可用内存  376MB → 721MB
```

## 附：诊断用的关键命令

```bash
# 回合是否卡住
zstd -d -c <session>.jsonl.zstd | grep -E 'turn/start|turn/end' | tail

# 进程是否在 D 状态
ps -eo pid,rss,stat,comm --sort=-rss | head

# 唤醒原因分布
journalctl -u qq-bridge | grep -oP '唤醒原因 group:\d+（\K[^）]+' | sort | uniq -c

# dsh 响应速度
curl -s -o /dev/null -w '%{http_code} %{time_total}s' http://127.0.0.1:3080/
```
