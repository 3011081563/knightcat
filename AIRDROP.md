# KNIGHTCAT 空投方案（1% = 10,000,000 枚）

## 一、为什么用 Merkle 白名单，而不是直接批量转账

| 方式 | 谁付 gas | 名单能改吗 | 适合 |
|---|---|---|---|
| **直接批量转账** | 你（每笔 ~51k gas，1000 人 ≈ 0.0025 BNB ≈ $1.8） | 不能，发出去就收不回 | 名单固定、人数少（<200） |
| **Merkle 领取**（本方案） | **领取者自己付**（~0.000003 BNB/人） | 能，改名单只改 claims.json（但 root 变了要重部署） | 人数多、要"互动感"、想让用户主动来领 |

Merkle 方式的核心：合约只存一个 **32 字节的 root**，谁是白名单、领多少，全靠 proof 现场验证。10 万人和 10 人，链上成本一样。

## 二、已交付的东西（都验证过）

| 组件 | 文件 | 状态 |
|---|---|---|
| 空投合约 | [`contracts/KNIGHTCATAirdrop.sol`](contracts/KNIGHTCATAirdrop.sol) | ✅ 编译通过，部署 gas ≈ **506,600**（≈$0.02） |
| 端到端测试 | [`test/evm-airdrop.js`](test/evm-airdrop.js) | ✅ **12/12 通过**（本地 EVM 真部署真领取） |
| 白名单生成器 | [`scripts/gen-airdrop.js`](scripts/gen-airdrop.js) | ✅ 已试跑，5 个地址 → root 正确 |
| Merkle 库 | [`scripts/merkle.js`](scripts/merkle.js) | ✅ 与合约同一套算法（keccak256 + 排序双哈希） |
| 领取页 | [`site/airdrop.html`](site/airdrop.html) | ✅ 已上线 |

**领取页（已上线）**：https://3011081563.github.io/knightcat/airdrop.html

测试覆盖的场景：

```
✅ 合约内余额正确（10,000,000）
✅ 白名单地址领取成功，到账数量精确
✅ 重复领取 → revert
✅ 篡改金额 → invalid proof → revert
✅ 别人代领 → not your claim → revert
✅ 领取后剩余额度正确
```

## 三、完整操作流程（6 步）

### ① 决定名单

1% = **10,000,000 枚**。三种常见分法：

| 方式 | 说明 |
|---|---|
| 早期买家 | 建池后前 N 名买入者（从 BscScan 交易记录导出） |
| 持币快照 | 某个区块高度持仓 > X 的地址，按持仓比例分 |
| 任务奖励 | 关注 X / 进 TG / 填表，收集地址后统一发 |

准备一个文件，两种格式都支持：

**recipients.json**
```json
[
  {"address":"0x1111...","amount":"3000000"},
  {"address":"0x2222...","amount":"2500000"}
]
```

**recipients.csv**
```
0x1111...,3000000
0x2222...,2500000
```

或者用权重自动缩放（总量固定 1%）：
```bash
node scripts/gen-airdrop.js recipients.csv --total-pct 1
```

### ② 生成 Merkle 根

```bash
node scripts/gen-airdrop.js recipients.json
```

输出：

```
════════ 空投白名单已生成 ════════
人数        : 5
空投总量    : 10000000.0 (1.0000%)
Merkle Root : 0x762449cc4a952e68037ac4dcb0ae5a1981ba5606d2d9da156ee3d5579dba57fc
输出        : airdrop/claims.json, airdrop/summary.json
```

### ③ 部署空投合约

构造函数三个参数：

| 参数 | 值 |
|---|---|
| `_token` | `0x578ec6134640efe429e8a2bfcdc13271b08d618b` |
| `_root` | 上一步打印的 Merkle Root |
| `_deadline` | 截止时间（unix 秒），比如 2027-01-01 = `1798761600` |

Remix 里选 `KNIGHTCATAirdrop`，Compiler 0.8.26 / optimize 200 / **EVM cancun**，填参数部署。
gas ≈ 506,600 → **0.000025 BNB ≈ $0.02**。

### ④ 把 1% 代币转进空投合约

```
transfer(空投合约地址, 10000000 * 1e18)
```
在 BscScan 的 Write Contract 里调你代币的 `transfer`，或用 MetaMask 直接发。

⚠️ 这一步不可逆：币进了合约，只有白名单地址领取才能出去。

### ⑤ 更新网站

1. 打开 [`site/airdrop.html`](site/airdrop.html)，把 `const AIRDROP = "0x0000…"` 改成真实地址
2. 把生成的 `airdrop/claims.json` 覆盖 `site/claims.json`
3. 推到 GitHub（`gh-pages` 分支），1 分钟后生效
4. 地址：`https://3011081563.github.io/knightcat/airdrop.html`

### ⑥ 推广

落地页加入口按钮 + 推特发一条："1% 供应量空投，白名单地址到官网领取，gas 自付"。

## 四、成本汇总

| 项 | 成本 |
|---|---|
| 部署空投合约 | 0.000025 BNB ≈ **$0.02** |
| 转入代币 | 0.000003 BNB ≈ $0.002 |
| 每个用户领取 | 用户自付 ≈ 0.000003 BNB |
| **项目方总投入** | **≈ $0.03 + 空投的代币本身** |

## 五、两个必须提前想清楚的坑

**① 没领完的币会永久锁死**
合约没有 owner、没有提现函数——这是"没有后门"的代价。想回收剩余额度就得加 owner，但那又变成"有后门"。
→ 建议：deadline 设长一点（6~12 个月），或者按"确定会领的人数"精确转账，别多转。

**② 白名单一旦部署就不能改**
root 写死在 immutable 变量里。要加人 = 重新部署一个新合约（又是 0.000025 BNB，很便宜）。

## 六、常用命令

```bash
# 生成白名单
node scripts/gen-airdrop.js recipients.json --total-pct 1

# 本地跑一遍完整测试（不花钱）
node test/evm-airdrop.js

# 看当前名单进度
cat airdrop/summary.json
```
