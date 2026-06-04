# Solana 金狗极速狙击机器人 (V0.63) — 技术预览

> ⚠️ **本仓库仅展示架构思路与部分脱敏代码片段，用于技术交流与引流。**
> 完整可商用版本（含完整策略参数、实盘交易逻辑、风控调优）**不公开**。
> 如需获取完整版，请联系作者。

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Sniper Bot V0.63                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ 扫描层        │───▶│  风控层       │───▶│  执行层      │  │
│  │  gmgn_*       │    │ check_safety │    │ jupiter_swap │  │
│  │  TLS 伪装     │    │ Mint/Freeze  │    │  5 重出局    │  │
│  │  Cloudflare   │    │ Top10/老鼠仓 │    │  追踪止损    │  │
│  │  穿透         │    │ LP 锁定检查  │    │  保本逃生    │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                    │                    │         │
│         ▼                    ▼                    ▼         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           监控层（每 5 秒询价，独立线程）             │    │
│  │  get_real_value / get_token_price_robust            │    │
│  │  双通道：lite-api（主力）+ quote-api（兜底）         │    │
│  └─────────────────────────────────────────────────────┘    │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           持久层（重启自动恢复）                       │    │
│  │  trade_history.csv  ← 历史账本                         │    │
│  │  active_positions.json ← 未平仓缓存                    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ✨ 核心模块概览

### 1️⃣ 极速扫描层（TLS 伪装 + Cloudflare 穿透）

**思路**：传统 Python 异步请求会被 Cloudflare 拦截。本项目通过底层 TLS 指纹伪装，让请求看起来像真实浏览器。

```python
# 仅展示思路：连接器配置（完整实现含 UA 轮换、Cookie 池等不在此公开）
class ForceIPv4Resolver(DefaultResolver):
    """强制 IPv4 解析，规避部分 VPS 的 IPv6 解析失败问题"""
    async def resolve(self, host, port=0, family=socket.AF_INET):
        return await super().resolve(host, port, family)

# 🚨 关键点：aiohttp TCPConnector 配置（已脱敏）
# - resolver=ForceIPv4Resolver()    # DNS 强制 IPv4
# - family=socket.AF_INET            # 协议族
# - ssl=False                        # ⚠️ 仅对非敏感接口；交易接口仍走 wss 加密
# - limit=0                          # 禁用连接池上限（高频场景需要）
```

> 🔒 **完整版含**：UA 池、Cookie 复用、失败自动重试退避策略、请求签名防风控细节。

---

### 2️⃣ 风控雷达 `check_safety`（防貔貅 / 防老鼠仓 / 防跑路）

**核心逻辑**：在买入前对代币做全方位体检，任一不通过直接枪毙。

```python
# 🚨 已脱敏：仅展示检测维度，具体阈值与权重由 CONF 配置
async def check_safety(self, mint: str) -> tuple:
    """
    返回: (是否通过, 原因, 推测symbol)
    """
    # ① 链上权限检查：Mint / Freeze 权限是否已丢弃
    #    - struct.unpack('<I', data[0:4])[0]  → Mint Authority
    #    - struct.unpack('<I', data[46:50])[0] → Freeze Authority
    #    任一非 0 直接拒绝（防貔貅盘）
    
    # ② 持仓集中度：前 10 大地址占比
    #    top10 / total_supply > 阈值  → 拒绝（防老鼠仓）
    
    # ③ Pump 盘智能识别（不依赖 mint 后缀，更稳健）
    #    dex_id 含 'pump' 或 labels 含 'pump' → 视为 pump 盘
    #    pump 盘用更低流动性门槛（$2000），普通盘用更高门槛
    
    # ④ 流动性池规模（防归零）
    # ⑤ 24h 交易量（防死盘）
    # ⑥ 买卖盘比率（卖盘太重直接毙）
    # ⑦ 社交媒体存在性（项目方态度）
    # ⑧ 代币年龄窗口（太新 = 风险，太老 = 错过最佳狙击点）
    
    return True, "✅ 这里的风景不错", symbol
```

> 🔒 **完整版含**：每项阈值的科学取值方法、Pump 盘专项策略、内部老鼠仓地址黑名单库。

---

### 3️⃣ 双通道实时询价（5 秒询价，毫秒级响应）

**亮点**：单一询价源可能挂，本项目实现**双通道自动降级**。

```python
# 🚨 已脱敏：仅展示双通道降级思路，具体超时与重试参数不在此公开
async def get_real_value(self, mint: str, raw_amount) -> float:
    """实时询价（lite-api 主力 + quote-api 兜底）"""
    
    # ── 通道 1：极速 lite-api 主力通道 ──
    # 优势：延迟低，1-2 秒返回
    # 风险：偶发返回异常数据
    try:
        # 构造 quote 请求（inputMint=代币, outputMint=WSOL, amount=持仓量）
        url = f"https://lite-api.jup.ag/swap/v1/quote?..."
        async with self.session.get(url, timeout=5) as r:
            if r.status == 200 and "outAmount" in (data := await r.json()):
                return float(data["outAmount"]) / 1e9  # SOL 9 位精度
    except Exception:
        pass  # 静默失败，自动降级
    
    # ── 通道 2：quote-api 备用通道（终极兜底） ──
    # 优势：稳定性高，主流通用
    # 劣势：延迟稍高
    try:
        alt_url = f"https://quote-api.jup.ag/v6/quote?..."
        async with self.session.get(alt_url, timeout=5) as r2:
            ...
    
    return None  # 双通道都失败
```

> 🔒 **完整版含**：Jupiter API Key 池轮换、异常返回值的 7 种边界处理、Decimal 精度无损转换。

---

### 4️⃣ 5 重智能出局策略（核心商业逻辑，**完全保密**）

> 🔒 **本节仅列出维度，具体阈值、优先级、冲突处理逻辑均为商业机密。**

| # | 策略 | 触发场景 | 作用 |
|---|------|---------|------|
| 1 | **静态止盈** | 涨到目标价位 | 自动分批回本 |
| 2 | **静态止损** | 跌到止损位 | 果断割肉 |
| 3 | **移动追踪止损** | 创新高后回撤 | 保护已得利润 |
| 4 | **保本逃生** | 涨过成本线后跌回 | 至少不亏 |
| 5 | **超时强制平仓** | 持仓超过时间窗口 | 防归零 |

**策略冲突时的优先级裁决算法**（付费版特有）：
- 多个策略同时触发时如何裁决？
- 部分成交如何处理（链上交易的非原子性）？
- 卖出失败时如何"逃生通道"放大滑点？

---

### 5️⃣ 记忆恢复机制（重启不掉链子）

```python
# 🚨 已脱敏：仅展示持久化文件与加载逻辑，具体字段处理略
def save_active_positions():
    """每次持仓变动都同步落盘"""
    with open("active_positions.json", "w") as f:
        json.dump(self.positions, f, indent=2)

def load_active_positions():
    """启动时自动加载未平仓币"""
    if os.path.exists("active_positions.json"):
        with open("active_positions.json", "r") as f:
            self.positions = json.load(f)
        log_console(f"📂 已恢复 {len(self.positions)} 个未平仓监控", Fore.GREEN)
```

**账本系统**：
- `trade_history.csv`：所有历史成交，按 mint 索引防止重复买入
- `active_positions.json`：实时持仓快照
- 重启后脚本自动接管监控，**绝不会"忘记"你正在盯的币**

---

## 🛡️ 双模切换：模拟盘 / 实盘

```python
# 一个开关切换，真钱假钱都用同一套代码
if CONF.get("DRY_RUN", False):
    # 🟢 模拟盘：只打日志、模拟发 TG，自带 1% 模拟损耗
    #    适合：新手跑 1-2 天熟悉出局逻辑
    log_trade_dry_run(...)
else:
    # 🔴 实盘：真实调用 Jupiter 消耗 SOL 上链
    await jupiter_swap(...)
```

**新手建议**：先 `DRY_RUN=True` 跑 1-2 天，看清楚机器人每一步在干什么，再切实盘。

---

## 🔧 技术栈

| 类别 | 选型 |
|------|------|
| 语言 | Python 3.10+ (asyncio) |
| 异步 HTTP | aiohttp + 强制 IPv4 |
| Solana SDK | solana-py + solders |
| DEX 聚合 | Jupiter (lite-api + quote-api) |
| 数据源 | GMGN / DexScreener |
| 通知 | Telegram Bot |
| 持久化 | CSV 账本 + JSON 缓存 |

---

## 📊 性能指标

- ⏱️ **扫描延迟**：< 2 秒（穿透 Cloudflare 后）
- ⏱️ **询价频率**：5 秒/次（每持仓独立线程）
- ⏱️ **从发现到下单**：< 1 秒（含风控 + Jupiter 报价）
- 📈 **并发持仓**：无上限（受 RPC 节点限流影响）

---

## 📜 完整版说明

本仓库仅展示**架构思路与脱敏代码片段**。

完整版包含：
- ✅ 所有策略的具体阈值与权重调优方法
- ✅ 完整的 Jupiter 交易实现（签名、滑点、重试）
- ✅ 5 重出局策略的冲突裁决算法
- ✅ Pump 盘专项狙击策略
- ✅ 一对一部署支持 + 7 天问题解答
- ✅ 后续版本免费更新

**完整版仅供私下交易，不在任何公开渠道分发。**

如有兴趣，请联系：
- GitHub: [@hs365](https://github.com/hs365)

---

## ⚠️ 免责声明

本项目仅供学习研究区块链自动化技术使用。
- 加密货币交易具有极高风险，**DYOR**（Do Your Own Research）
- 模拟盘跑通不代表实盘能赚钱
- 因使用本项目产生的任何盈亏，与作者无关
- 请遵守当地法律法规

---

_© 2026 hs365 · All Rights Reserved_
