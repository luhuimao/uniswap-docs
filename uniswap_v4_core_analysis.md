# Uniswap v4-core 代码深度分析

> 目录：`briefcase/protocols/v4-core`

---

## 1. 整体目录结构

```
v4-core/
├── interfaces/        # 核心接口定义
│   ├── IPoolManager.sol      # 最核心接口：PoolManager 全部操作
│   ├── IHooks.sol            # Hook 回调接口
│   ├── IProtocolFees.sol     # 协议费相关
│   ├── IExtsload.sol         # 外部 sload 访问
│   ├── IExttload.sol         # 外部 tload（瞬态存储）访问
│   └── external/
│       └── IERC6909Claims.sol # ERC-6909 多代币标准
├── libraries/         # 核心逻辑库
│   ├── Pool.sol              # 池子状态机（最核心）
│   ├── Hooks.sol             # Hook 调度与权限
│   ├── SwapMath.sol          # 单步 swap 数学
│   ├── SqrtPriceMath.sol     # sqrt 价格数学
│   ├── TickMath.sol          # Tick ↔ sqrtPrice 转换
│   ├── TickBitmap.sol        # Tick 位图（快速查找下一个初始化 tick）
│   ├── Position.sol          # LP 仓位状态与费用计算
│   ├── StateLibrary.sol      # 外部读取池子存储（extsload）
│   ├── LPFeeLibrary.sol      # LP 费率工具
│   ├── ProtocolFeeLibrary.sol# 协议费率工具
│   ├── FullMath.sol          # 512位精度乘除法
│   ├── SafeCast.sol          # 安全类型转换
│   ├── CurrencyDelta.sol     # 瞬态存储中的余额 delta
│   ├── CurrencyReserves.sol  # 货币准备金追踪
│   ├── Lock.sol              # 全局锁状态
│   ├── NonzeroDeltaCount.sol # 未结清 delta 计数
│   └── ...                   # 其他工具库
└── types/             # 自定义值类型（节省 gas）
    ├── PoolKey.sol           # 池子唯一标识（5元组）
    ├── PoolId.sol            # PoolKey 的 keccak256 hash
    ├── Slot0.sol             # 池子核心状态（打包 bytes32）
    ├── BalanceDelta.sol      # 双代币余额变化量（int256 打包）
    ├── BeforeSwapDelta.sol   # beforeSwap hook 返回值
    ├── Currency.sol          # 货币地址（0x0 代表 ETH）
    └── PoolOperation.sol     # 操作参数结构体
```

---

## 2. 核心架构：Flash Accounting（闪电记账）

v4 最重大的架构革新是 **单合约 + Flash Accounting** 模型。

### 2.1 Unlock / Lock 模型

```
调用者 ─── unlock(data) ──▶ PoolManager
                               │
                               ├── 设置全局 lock = UNLOCKED
                               │
                               └── callback: IUnlockCallback(msg.sender).unlockCallback(data)
                                          │
                                          ├── swap(...)        ←┐
                                          ├── modifyLiquidity(...)│  在 callback 内可调用
                                          ├── settle()         ←┘
                                          └── take(...)
                               │
                               └── 检查所有 delta 已归零（nonzeroDeltaCount == 0）
```

**核心思路**：
- 所有改变余额的操作（swap、addLiquidity 等）都必须在 `unlock` 的回调函数 `unlockCallback` 内调用
- 每次操作只记录**差值（delta）**到瞬态存储（transient storage，EIP-1153）
- 回调结束前，调用者必须通过 `settle()`（存入代币）或 `take()`（取出代币）将所有 delta 清零
- 若 `unlockCallback` 结束时存在未结清的 `nonzeroDeltaCount`，交易 revert

**好处**：
- 天然支持闪电贷（flash loan）：先 `take()` 拿走代币，用完后 `settle()` 还回即可
- 多池复合操作只需一次 ERC20 transfer，极大节省 gas
- 通过 `CurrencyDelta`（transient storage）追踪每个调用者对每种货币的净 delta

### 2.2 Token 结算方式

| 方法 | 说明 |
|------|------|
| `sync(currency)` | 快照当前 ERC20 余额到瞬态存储（ERC20转账前必须调用） |
| `settle()` | 用已转入合约的代币结清欠款（ERC20用 sync+transfer+settle，ETH 直接 payable） |
| `settleFor(recipient)` | 代他人结算 |
| `take(currency, to, amount)` | 从池子取出代币（相当于借出） |
| `mint(to, id, amount)` | 将余额铸造成 ERC6909 凭证（无需实际转账） |
| `burn(from, id, amount)` | 销毁 ERC6909 凭证来结算欠款 |
| `clear(currency, amount)` | 放弃正 delta（视为粉尘） |

---

## 3. Hooks 系统

### 3.1 地址编码权限（Address-Encoded Permissions）

**v4 的革命性设计**：Hook 合约需要部署到特定地址，地址的**最低14位** 决定它拥有哪些 hook 权限：

```
bit 13: BEFORE_INITIALIZE
bit 12: AFTER_INITIALIZE
bit 11: BEFORE_ADD_LIQUIDITY
bit 10: AFTER_ADD_LIQUIDITY
bit  9: BEFORE_REMOVE_LIQUIDITY
bit  8: AFTER_REMOVE_LIQUIDITY
bit  7: BEFORE_SWAP
bit  6: AFTER_SWAP
bit  5: BEFORE_DONATE
bit  4: AFTER_DONATE
bit  3: BEFORE_SWAP_RETURNS_DELTA   ← Hook 可修改 swap 数量
bit  2: AFTER_SWAP_RETURNS_DELTA    ← Hook 可修改 swap 数量
bit  1: AFTER_ADD_LIQUIDITY_RETURNS_DELTA
bit  0: AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA
```

**举例**：地址 `0x...2400` 末位为 `10 0100 0000 0000` → 启用 `BEFORE_INITIALIZE`（bit13）和 `AFTER_ADD_LIQUIDITY`（bit10）。

这要求 hook 合约使用 `CREATE2` + salt 暴力算出符合权限的地址（类似 vanity address 挖矿）。

### 3.2 Hook 调用流程（以 Swap 为例）

```
IPoolManager.swap()
    │
    ├── Hooks.beforeSwap()
    │     ├── 检查 BEFORE_SWAP_FLAG
    │     ├── call IHooks.beforeSwap(sender, key, params, hookData)
    │     ├── 解析返回的 BeforeSwapDelta（可修改 amountSpecified）
    │     └── 解析动态 LP fee 覆盖值
    │
    ├── Pool.swap()  ← 核心 AMM 逻辑
    │
    └── Hooks.afterSwap()
          ├── 检查 AFTER_SWAP_FLAG
          ├── call IHooks.afterSwap(sender, key, params, swapDelta, hookData)
          ├── 若 AFTER_SWAP_RETURNS_DELTA_FLAG：hook 可返回 int128 调整未指定侧金额
          └── 计算最终 callerDelta 和 hookDelta
```

### 3.3 Hook Delta 机制

带 `RETURNS_DELTA` 标志的 hook 可以直接参与资金流：
- **正 delta**：hook 从调用者那里拿走货币（hook owes/sent currency）
- **负 delta**：hook 向调用者提供货币（hook is owed/took currency）

这使 Hook 能实现：
- 自定义再平衡逻辑
- 动态路由
- 订单薄混合流动性
- 费用分成给第三方

---

## 4. 池子状态管理（Pool.sol + Slot0.sol）

### 4.1 Pool.State 结构

```solidity
struct State {
    Slot0 slot0;                                        // 核心价格状态（1 slot，32字节）
    uint256 feeGrowthGlobal0X128;                       // token0 全局费用累积
    uint256 feeGrowthGlobal1X128;                       // token1 全局费用累积
    uint128 liquidity;                                  // 当前价格区间的活跃流动性
    mapping(int24 => TickInfo) ticks;                   // 每个 tick 的数据
    mapping(int16 => uint256) tickBitmap;               // tick 初始化位图
    mapping(bytes32 => Position.State) positions;       // 仓位数据
}
```

### 4.2 Slot0 极致打包（节省 gas）

```
256 bits total:
┌──────────────┬────────┬─────────────────┬────────┬────────────────────────────────┐
│  24bits(空)  │lpFee24 │ protocolFee(24) │tick(24)│    sqrtPriceX96 (160bits)      │
└──────────────┴────────┴─────────────────┴────────┴────────────────────────────────┘
 bit 255..232   231..208     207..184       183..160         159..0
```

所有字段用纯 assembly 读写，一次 SLOAD 获取所有关键状态，节省 gas。

### 4.3 PoolKey：池子的唯一标识

```solidity
struct PoolKey {
    Currency currency0;   // 地址较小的代币（address(0) = ETH 原生支持）
    Currency currency1;   // 地址较大的代币
    uint24 fee;           // LP 费率（0x800000 代表动态费率）
    int24 tickSpacing;    // Tick 间距
    IHooks hooks;         // Hook 合约地址（同时编码权限）
}
```

`PoolId = keccak256(abi.encode(PoolKey))`，相同参数的池子全局唯一。

**注意**：v4 中 `hooks` 是 PoolKey 的一部分，同样货币对、同样费率、但 hook 不同，就是完全不同的池子。

---

## 5. Swap 算法（Pool.sol）

### 5.1 核心循环：跨 Tick 迭代

```
while (amountRemaining != 0 && price != sqrtPriceLimitX96):
    1. 从 tickBitmap 找下一个初始化的 tick（单 word 内）
    2. 计算到 nextTick 的 sqrtPriceNextX96
    3. SwapMath.computeSwapStep() 计算本步：
       - amountIn（消耗输入）
       - amountOut（产生输出）
       - feeAmount（LP 费用）
       - 新的 sqrtPrice
    4. 扣除 protocol fee，剩余 fee 累积到 feeGrowthGlobalX128
    5. 若到达 nextTick：
       - 若 tick initialized → crossTick()（翻转 feeGrowthOutside，更新流动性）
       - liquidityNet 正/负（zeroForOne时取反）调整 result.liquidity
    6. 更新 tick index（zeroForOne: tick = tickNext - 1，防止边界问题）
```

### 5.2 Amount 符号约定

| amountSpecified | 含义 |
|---|---|
| **负数** | exactInput（指定输入量）|
| **正数** | exactOutput（指定输出量）|

### 5.3 费率计算

```
totalSwapFee = protocolFee > 0
    ? protocolFee.calculateSwapFee(lpFee)    // 叠加计算
    : lpFee

protocolFeeAmount = (amountIn + feeAmount) * protocolFee / 1_000_000
lpFeeAmount = feeAmount - protocolFeeAmount
feeGrowthGlobal += lpFeeAmount * Q128 / liquidity
```

---

## 6. 流动性仓位（Position.sol）

### 6.1 Position Key

```solidity
positionKey = keccak256(abi.encodePacked(owner, tickLower, tickUpper, salt))
```

`salt` 是 v4 新增字段，同一 owner 可在同一 tick 范围内持有多个不同仓位。

### 6.2 费用计算（Snapshot 机制）

```
feesOwed0 = liquidity × (feeGrowthInside0X128 - feeGrowthInside0LastX128) / Q128
feesOwed1 = liquidity × (feeGrowthInside1X128 - feeGrowthInside1LastX128) / Q128
```

`feeGrowthInside` 由 tick 边界的 `feeGrowthOutside` 差分计算：
- 当前 tick 在区间内：`feeGrowthGlobal - lower.outside - upper.outside`
- 当前 tick 在区间下方：`lower.outside - upper.outside`
- 当前 tick 在区间上方：`upper.outside - lower.outside`

---

## 7. 费率系统（LPFeeLibrary.sol）

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_LP_FEE` | `1_000_000` | 最大静态 LP 费率（100% = 1_000_000 pips）|
| `DYNAMIC_FEE_FLAG` | `0x800000` | 最高位置1 → 动态费率池 |
| `OVERRIDE_FEE_FLAG` | `0x400000` | beforeSwap hook 可覆盖当次 swap 的费率 |

**动态费率池**：
- 初始费率为 0
- Hook 在 `afterInitialize` 中设置初始费率
- `beforeSwap` 返回值第23位置1 + 合法费率值 → 覆盖本次 swap 费率
- 任何时候 hook 可调用 `PoolManager.updateDynamicLPFee()` 更新全局 LP 费率

---

## 8. StateLibrary：外部读取池子状态

```solidity
// 利用 IExtsload 接口直接从外部读取 PoolManager 内部 storage
// POOLS_SLOT = 6  (pools mapping 在 PoolManager storage 中的 slot)
bytes32 stateSlot = keccak256(abi.encodePacked(poolId, POOLS_SLOT));

// 读取 Slot0
bytes32 data = manager.extsload(stateSlot);

// 读取 TickInfo（批量 extsload 3个 word）
bytes32[] memory data = manager.extsload(tickInfoSlot, 3);

// 读取 Position
bytes32[] memory data = manager.extsload(positionSlot, 3);
```

`IExtsload` / `IExttload` 是 v4 独特接口，允许外部合约通过 `extsload(slot)` 和 `exttload(slot)` 读取 PoolManager 的 storage 和 transient storage，避免重复实现视图函数，极大降低合约大小。

---

## 9. BalanceDelta：双代币 Delta 的高效打包

```solidity
type BalanceDelta is int256;
// 高128位 = amount0（int128）
// 低128位 = amount1（int128）
```

用 Solidity 用户自定义类型（UDT）+ 运算符重载实现，所有加减法用 assembly 完成，无需解包再打包，极省 gas。

---

## 10. 关键设计亮点总结

| 特性 | v3 | v4 |
|---|---|---|
| 架构 | 每个池子独立合约 | 所有池子在**单一 PoolManager** |
| Gas 效率 | 每次操作单独 transfer | **Flash Accounting**，多操作一次 transfer |
| 可扩展性 | 无 hook | **14种 Hook 点**，地址编码权限 |
| 费率 | 固定几档 | 任意静态费率 + **动态费率** |
| 原生 ETH | 需 WETH | **直接支持**（Currency(0) = ETH）|
| 仓位区分 | owner+tickRange | +**salt** 额外区分 |
| 状态读取 | 需 view 函数 | **IExtsload**（外部直读 storage）|
| 代币余额 | ERC20 only | ERC20 + **ERC6909** 内部凭证 |
| 瞬态存储 | ❌ | ✅ **EIP-1153 transient storage** |
