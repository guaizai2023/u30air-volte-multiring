# U30 Air VoLTE ↔ SIP · 多终端同时振铃版

> 上游单分机版本：[**guaizai2023/u30air-volte-sip**](https://github.com/guaizai2023/u30air-volte-sip)
> 本仓库是它的一个变体，**只改「来电时叫几台设备」这一件事**，其余完全相同。

---

## 为什么要单独开一版

上游版本把呼入链路完整打通了：外部打 SIM 号 → SIP 分机响 → 接听 → 双向语音 → 双向挂断同步。
但实测发现一个**客户端侧**的硬限制：

| 档位 | iPhone 锁屏状态下的结果 |
|---|---|
| T+15 分钟 | ✅ 响铃，1.5 秒 |
| T+52 分钟 | ✅ 响铃，1 秒 |
| **T+2 小时** | ❌ **不响**，解锁后立刻恢复 |

关键证据是：那通电话 Asterisk **照常发出了 INVITE**（`AMI originate -> ok queued`），
而分机的 TCP 连接**从头到尾没断过**（source port 一直是 `65311`），
只是 iOS 把 App 进程挂起了——socket 还挂在系统手里，连接"看起来在线"，App 却处理不了来电。

结论：**单一 iOS 终端无法长期充当座机**。Force Background 只能把命续到 1~2 小时。
PortSIP 那种能锁屏收来电的推送，要求服务端是 PortSIP PBX；我们跑的是 Asterisk，发不了 APNs。

所以这版做的事很简单：**一个来电，同时叫多台终端**。哪台先接算哪台，其余自动挂断。
加一台便宜的 Android 手机或桌面 IP 话机当主力，iPhone 当移动分机，掉线也不影响接电话。

---

## 改了什么

对比上游，只有两处。

### 1. `tools/mu300-volte-in.py`

分机从一个变成一组（环境变量 `MU300_IN_EXTS`，逗号分隔）：

```python
EXTS = [x.strip() for x in os.environ.get(
    "MU300_IN_EXTS", os.environ.get("MU300_IN_EXT", "6001")).split(",") if x.strip()]
```

发起呼叫时按数量分支：

```python
if len(EXTS) == 1:
    # 单个分机：直接叫它，接听后立刻进 AudioSocket
    originate("PJSIP/" + EXTS[0], "AudioSocket", AUDIOSOCKET)
else:
    # 多个分机：交给 dialplan 的 Dial(a&b)，由 Asterisk 原生做多路振铃
    originate("Local/s@volte-in", "Wait", "90")
```

单分机时走的还是上游那条最短路径，**行为完全不变**，所以这版对单终端用户是零成本兼容的。

### 2. `etc/asterisk/extensions.conf.example`

```asterisk
[volte-in]
exten => s,1,NoOp(VoLTE incoming, multiring)
 same => n,Dial(PJSIP/6001&PJSIP/6002,25)
 same => n,GotoIf($["${DIALSTATUS}" = "ANSWER"]?answered:missed)
 same => n(answered),NoOp(extension answered, start audio bridge)
 same => n,AudioSocket(6d7f8a3e-4bcd-4e2f-9a01-0123456789ab,127.0.0.1:9092)
 same => n,Hangup()
 same => n(missed),NoOp(no answer, hang up VoLTE)
 same => n,System(mu300-at -t 6 "AT+CHUP")
 same => n,Hangup()
```

加设备就改 `Dial(...)` 那一行，每个被叫都要带技术前缀，用 `&` 连接。

---

## 原理：为什么用 `Dial(a&b)` 而不是发两次 Originate

Asterisk 的 `Dial()` 支持一次拨多个目的地，这是**内建**语义：

- 同时向所有被叫发 INVITE
- **第一个应答的胜出**，其余自动 CANCEL
- 返回 `${DIALSTATUS}`（`ANSWER` / `NOANSWER` / `BUSY` / `CHANUNAVAIL`）

比"对每个分机各发一次 AMI Originate、再自己判断谁先接"要可靠得多——
不用处理竞态、不用手动挂断落败的通道、也不会出现两路同时接进来抢音频桥。

代价是要绕一层 `Local` 通道：AMI 发起 `Local/s@volte-in`，这条通道的一半进拨号计划执行 `Dial`，
另一半执行 `Wait`。这也是本版**唯一需要实测确认**的地方（见下）。

---

## 部署

与上游相同的部分不再重复，只列差异。

```sh
# 1. 分机名单：改 init 脚本里的默认值，或直接给环境变量
#    /etc/init.d/mu300-volte-in
#    RING_EXTS=${MU300_IN_EXTS:-6001,6002}

# 2. 拨号计划：Dial() 里列出同样的分机
scp etc/asterisk/extensions.conf.example root@192.168.77.1:/etc/asterisk/extensions.conf
ssh root@192.168.77.1 'asterisk -rx "dialplan reload"'

# 3. 守护进程
scp tools/mu300-volte-in.py   root@192.168.77.1:/usr/bin/
scp etc/init.d/mu300-volte-in root@192.168.77.1:/etc/init.d/
ssh root@192.168.77.1 'chmod +x /usr/bin/mu300-volte-in.py /etc/init.d/mu300-volte-in
                       /etc/init.d/mu300-volte-in restart'
```

**两处分机名单必须一致**（环境变量 + `Dial()`），否则会出现"某台永远不响"。

---

## 待验证

| 项目 | 状态 | 说明 |
|---|---|---|
| `Local/s@volte-in` 能正确触发 dialplan | ⬜ **未实测** | 本版的关键假设。若不通，退回"逐个 Originate + 手动挂断落败通道" |
| 两台终端同时振铃 | ⬜ **未实测** | 需要第二台设备参与 |
| 先接先通、其余自动挂断 | ⬜ **未实测** | 依赖 `Dial()` 内建行为，理论可靠 |
| 单分机模式行为不变 | ⬜ 待回归 | 应完全等同上游 |
| 其余（音频、挂断同步、超时） | ✅ 继承上游 | 见上游 `docs/03-inbound-call.md` |

> 本仓库的代码是**照着 Asterisk 语义写的，但尚未在真机上跑通多终端路径**。
> 单分机路径与上游逐字相同，可以直接用。

---

## 和上游的关系

- 呼出、音频桥、DSP、延迟处理、挂断同步 —— **一行没改**，全部来自上游。
- 上游那版是验证过的稳定版；这一版是**待验证的实验分支**。
- 如果多终端不是你的需求，用上游版就好，别用这个。
