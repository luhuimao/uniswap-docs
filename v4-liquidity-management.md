# Uniswap v4 流动性池管理详解

---

## 一、池子的创建与标识

### PoolKey —— 池子的"身份证"

```solidity
struct PoolKey {
    Currency currency0;   // 地址较小的代币（address(0) = 原生 ETH）
    Currency currency1;   // 地址较大的代币
    uint24 fee;           // LP 费率 / 0x800000 表示动态费率
    int24 tickSpacing;    // Tick 最小间距
    IHooks hooks;         // Hook 合约地址（同时编码14位权限）
}
```

`PoolId = keccak256(abi.encode(PoolKey))`，五个参数组合唯一标识一个池子。**hooks 地址是 PoolKey 的一部分**，同一对代币、相同费率、但 Hook 不同 → 完全不同的池子，可以同时并存。

### 池子初始化 `initialize()`

```
调用者 → PoolManager.initialize(key, sqrtPriceX96)
    ↓
1. 验证 PoolKey（货币顺序、tickSpacing 范围、Hook 地址合法性）
2. 计算初始 tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96)
3. Pool.initialize():
   - 检查 slot0.sqrtPriceX96 == 0（防止重复初始化）
   - 写入 slot0: sqrtPriceX96 + tick + lpFee（protocolFee 初始为 0）
4. 调用 Hooks.beforeInitialize / afterInitialize
5. emit Initialize 事件
```

---

## 二、Pool.State —— 池子的全部状态

每个池子在 `PoolManager` 的 `mapping(PoolId => Pool.State) pools` 中各有一个状态对象：

```solidity
struct State {
    Slot0 slot0;                                          // ① 核心价格状态（1个slot，32字节）
    uint256 feeGrowthGlobal0X128;                         // ② token0 累计费用（全局，单调递增）
    uint256 feeGrowthGlobal1X128;                         // ③ token1 累计费用
    uint128 liquidity;                                    // ④ 当前价格处的活跃流动性
    mapping(int24 tick => TickInfo) ticks;                // ⑤ 各 tick 点数据
    mapping(int16 wordPos => uint256) tickBitmap;         // ⑥ tick 初始化位图
    mapping(bytes32 posKey => Position.State) positions;  // ⑦ LP 仓位数据
}
```

### Slot0 打包布局（极致节省 SLOAD）

```
256 bits:
┌──────────┬──────────┬─────────────────┬──────────┬────────────────────────┐
│  24(空)  │ lpFee:24 │ protocolFee:24  │ tick:24  │   sqrtPriceX96:160     │
└──────────┴──────────┴─────────────────┴──────────┴────────────────────────┘
```

一次 SLOAD 读出当前价格、tick、两个费率，全部 Assembly 位操作读写。

---

## 三、添加流动性（modifyLiquidity + 正 liquidityDelta）

```
用户调用 PoolManager.modifyLiquidity(key, params, hookData)
params = { owner, tickLower, tickUpper, liquidityDelta(+), tickSpacing, salt }
```

### Step 1：校验 Tick 范围

```solidity
if (tickLower >= tickUpper) revert TicksMisordered
if (tickLower < MIN_TICK)   revert TickLowerOutOfBounds
if (tickUpper > MAX_TICK)   revert TickUpperOutOfBounds
```

### Step 2：更新两侧 Tick（updateTick）

对 `tickLower` 和 `tickUpper` 各调用一次 `updateTick`：

```
liquidityGrossAfter = liquidityGross + liquidityDelta

flipped = (liquidityGrossAfter == 0) != (liquidityGrossBefore == 0)
        → true 表示该 tick 从未初始化变为已初始化（或反过来）

若 tick 刚被初始化（liquidityGrossBefore == 0）：
    若 tick <= currentTick:
        feeGrowthOutside = feeGrowthGlobal（约定：初始时外侧 = 全局）
    否则:
        feeGrowthOutside = 0

liquidityNet:
    lower tick: liquidityNet += liquidityDelta（价格从左穿越时增加流动性）
    upper tick: liquidityNet -= liquidityDelta（价格从右穿越时减少流动性）

最后用一次 sstore 同时写 liquidityGross + liquidityNet（原子写入，节省 gas）
```

### Step 3：若 Tick 被 flip → 更新 TickBitmap

```solidity
tickBitmap.flipTick(tick, tickSpacing)
// 将 tick / tickSpacing 位置的 bit 取反（0→1 或 1→0）
```

位图结构：`mapping(int16 wordPos => uint256 word)`，每个 word 代表 256 个 tick。

### Step 4：检查每个 tick 的最大流动性上限

```solidity
maxLiquidityPerTick = type(uint128).max / numTicks  // 防止单 tick 流动性溢出
if (liquidityGrossAfter > maxLiquidityPerTick) revert TickLiquidityOverflow
```

### Step 5：计算区间内费用增长（getFeeGrowthInside）

```
若 currentTick < tickLower（价格在区间左侧）：
    feeGrowthInside0 = lower.feeGrowthOutside0 - upper.feeGrowthOutside0

若 currentTick >= tickUpper（价格在区间右侧）：
    feeGrowthInside0 = upper.feeGrowthOutside0 - lower.feeGrowthOutside0

若 tickLower <= currentTick < tickUpper（价格在区间内）：
    feeGrowthInside0 = feeGrowthGlobal0 - lower.feeGrowthOutside0 - upper.feeGrowthOutside0
```

### Step 6：更新 Position（计算已积累费用）

```solidity
// positionKey = keccak256(owner, tickLower, tickUpper, salt)
Position.State storage position = positions[positionKey];

// 已积累费用 = 持仓量 × (当前区间内累计 - 上次快照)
feesOwed0 = FullMath.mulDiv(
    feeGrowthInside0X128 - position.feeGrowthInside0LastX128,
    position.liquidity,
    Q128
);

// 更新仓位
position.liquidity += liquidityDelta;
position.feeGrowthInside0LastX128 = feeGrowthInside0X128;  // 更新快照
```

### Step 7：计算需要存入的 Token 数量（delta）

```
若 currentTick < tickLower（价格在区间左侧）：
    delta = (getAmount0Delta(sqrtLower, sqrtUpper, liquidityDelta), 0)
    → 只需存 token0（等待价格上升进入区间）

若 tickLower <= currentTick < tickUpper（价格在区间内）：
    delta = (getAmount0Delta(sqrtPrice, sqrtUpper, liquidityDelta),
             getAmount1Delta(sqrtLower, sqrtPrice, liquidityDelta))
    → 双侧存入，同时更新 pool.liquidity（活跃流动性）

若 currentTick >= tickUpper（价格在区间右侧）：
    delta = (0, getAmount1Delta(sqrtLower, sqrtUpper, liquidityDelta))
    → 只需存 token1（等待价格下降进入区间）
```

---

## 四、移除流动性（负 liquidityDelta）

流程与添加基本对称，额外一步：若 tick 被 flip（流动性从有变无），**清除 tick 数据**：

```solidity
if (flippedLower) clearTick(self, tickLower);   // delete ticks[tickLower]
if (flippedUpper) clearTick(self, tickUpper);
```

释放 storage 获得退款，节省长期存储成本。

---

## 五、Swap 时的流动性切换（crossTick）

Swap 过程中价格穿越 tick 边界时，动态切换活跃流动性：

```
价格穿越 tickNext（zeroForOne，价格向左移动）：

1. crossTick(tickNext):
   - 翻转 feeGrowthOutside（内外互换）：
     feeGrowthOutside0 = feeGrowthGlobal0 - feeGrowthOutside0
   - 返回 liquidityNet

2. 由于向左穿越，liquidityNet 取反：liquidityNet = -liquidityNet

3. 更新活跃流动性：
   result.liquidity = LiquidityMath.addDelta(result.liquidity, liquidityNet)

4. 更新 tick（向左需要 -1 修正）：
   result.tick = tickNext - 1
```

**liquidityNet 的方向约定**：
- 从左向右穿越 lower tick → 流动性增加（+liquidityDelta）
- 从左向右穿越 upper tick → 流动性减少（-liquidityDelta）
- 向右穿越时直接用，向左穿越时取反

---

## 六、Position 的 Salt 机制（v4 新增）

v4 中仓位 key 多了一个 `salt` 字段：

```
positionKey = keccak256(owner, tickLower, tickUpper, salt)
```

**作用**：同一个 owner、相同 tick 范围可以持有**多个独立仓位**，通过不同 salt 区分。

**应用场景**：
- 路由合约为不同用户在同一价格区间管理各自独立的仓位
- 同一用户在同一区间分批添加流动性，各批次费用独立结算
- Hook 合约为自己保留一个专用仓位（用合约地址作 salt）

---

## 七、流动性管理全景总结

```
Pool.State 的 liquidity（活跃流动性）变化时机：
  ① modifyLiquidity 且 currentTick ∈ [tickLower, tickUpper) → 直接 ±= liquidityDelta
  ② Swap crossTick → ±= liquidityNet

tick.liquidityGross（总被引用量）变化时机：
  → modifyLiquidity 时无条件更新（决定 tick 是否需要初始化/清除）

tick.liquidityNet（净变化量）变化时机：
  → modifyLiquidity 时更新（lower += delta，upper -= delta）
  → crossTick 时消费（决定 pool.liquidity 变化方向和大小）

feeGrowthGlobal（全局费用）变化时机：
  → 每一步 swap 后，按 feeAmount / liquidity 累积

feeGrowthOutside（tick 边界费用）变化时机：
  → tick 首次初始化（若 tick <= currentTick，设为 global）
  → crossTick 时翻转

position.feeGrowthInsideLast 变化时机：
  → modifyLiquidity（收割未结算费用后更新快照）
```

---

## 八、费用结算的数学原理（O(1) 计算）

```
feeGrowthInside（区间内费用增长）基于差分翻转原理：
  在区间内时：feeGrowthGlobal - lower.outside - upper.outside 持续增大
  tick 被穿越：outside 翻转，等效于之前的 inside 被"冻结"，新的 inside 重新从 0 累积

LP 收益 = liquidity × (feeGrowthInside_now - feeGrowthInside_lastSnapshot) / Q128
```

无论池子有多少个仓位、多少个 tick，任意仓位的费用计算只需读取：
- 1 次 `feeGrowthGlobal`（pool 级）
- 2 次 `feeGrowthOutside`（tick 级，lower + upper）
- 1 次 `feeGrowthInsideLast`（position 级）

**时间复杂度 O(1)**，与整个池子的规模完全无关。
