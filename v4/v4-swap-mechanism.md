# Uniswap v4 Swap 原理详解

---

## 一、入口与参数约定

```
PoolManager.swap(key, params, hookData)
params = {
    amountSpecified,      // 负数 = exactInput，正数 = exactOutput
    zeroForOne,           // true = token0换token1（价格向左移动）
    sqrtPriceLimitX96,    // 价格滑点保护上/下界
    tickSpacing,          // 继承自 PoolKey
    lpFeeOverride         // beforeSwap hook 可覆盖的费率
}
```

| `amountSpecified` | 方向 | 含义 |
|---|---|---|
| **负数** | 输入方 | exactInput：我最多给这么多 |
| **正数** | 输出方 | exactOutput：我至少要这么多 |

---

## 二、费率确定

```
// 1. 读取 beforeSwap hook 是否有覆盖费率（动态费率池专用）
lpFee = params.lpFeeOverride.isOverride()
    ? params.lpFeeOverride.removeOverrideFlagAndValidate()
    : slot0.lpFee()

// 2. 叠加协议费
protocolFee = zeroForOne
    ? slot0.protocolFee().getZeroForOneFee()   // 低12位
    : slot0.protocolFee().getOneForZeroFee()   // 高12位

swapFee = protocolFee == 0
    ? lpFee
    : uint16(protocolFee).calculateSwapFee(lpFee)
// 叠加公式：swapFee = protocolFee + lpFee - protocolFee * lpFee / 1e6
```

---

## 三、价格边界校验

```
zeroForOne（价格向左，sqrtPrice 减小）:
    sqrtPriceLimitX96 必须 < sqrtPriceCurrentX96
    sqrtPriceLimitX96 必须 > MIN_SQRT_PRICE

oneForZero（价格向右，sqrtPrice 增大）:
    sqrtPriceLimitX96 必须 > sqrtPriceCurrentX96
    sqrtPriceLimitX96 必须 < MAX_SQRT_PRICE
```

---

## 四、核心：跨 Tick 迭代循环

```
初始化：
    amountSpecifiedRemaining = params.amountSpecified
    amountCalculated = 0
    result.sqrtPriceX96 = slot0.sqrtPriceX96()
    result.tick = slot0.tick()
    result.liquidity = pool.liquidity
    step.feeGrowthGlobalX128 = feeGrowthGlobal0 或 1（取决于方向）

while (amountSpecifiedRemaining ≠ 0 AND result.sqrtPrice ≠ sqrtPriceLimit):
```

### Step 1：找下一个已初始化的 Tick

```solidity
(step.tickNext, step.initialized) =
    tickBitmap.nextInitializedTickWithinOneWord(
        result.tick, params.tickSpacing, zeroForOne
    );
// 只在当前 word（256个tick）内搜索，跨 word 时返回边界值
if (step.tickNext <= MIN_TICK) step.tickNext = MIN_TICK;
if (step.tickNext >= MAX_TICK) step.tickNext = MAX_TICK;

step.sqrtPriceNextX96 = TickMath.getSqrtPriceAtTick(step.tickNext);
```

### Step 2：计算本步交换量（SwapMath.computeSwapStep）

```
目标价格 = min/max(sqrtPriceNextX96, sqrtPriceLimitX96)

exactInput（amountSpecified < 0）:
    可用 = |amountRemaining| × (1_000_000 - feePips) / 1_000_000
    if 可用 >= maxAmountIn:
        sqrtPriceNext = sqrtTarget; amountIn = maxAmountIn
    else:
        sqrtPriceNext = getNextSqrtPriceFromInput(...)
        amountIn = 可用
    feeAmount = |amountRemaining| - amountIn
    amountOut = getAmount0/1Delta(sqrtStart, sqrtPriceNext, liquidity, roundDown)

exactOutput（amountSpecified > 0）:
    if amountRemaining >= maxAmountOut:
        sqrtPriceNext = sqrtTarget; amountOut = maxAmountOut
    else:
        sqrtPriceNext = getNextSqrtPriceFromOutput(...)
        amountOut = amountRemaining
    amountIn  = getAmount0/1Delta(..., roundUp)
    feeAmount = amountIn × feePips / (1_000_000 - feePips)
```

### Step 3：更新剩余量

```
exactInput:
    amountSpecifiedRemaining += (step.amountIn + step.feeAmount)  // 负数趋向 0
    amountCalculated += step.amountOut

exactOutput:
    amountSpecifiedRemaining -= step.amountOut  // 正数趋向 0
    amountCalculated -= (step.amountIn + step.feeAmount)
```

### Step 4：分配协议费用

```
if protocolFee > 0:
    delta = (swapFee == protocolFee)
        ? step.feeAmount                                          // LP费为0，全归协议
        : (step.amountIn + step.feeAmount) × protocolFee / 1e6   // 按比例分

    step.feeAmount -= delta     // LP 费减少
    amountToProtocol += delta   // 协议费累计
```

### Step 5：累积 LP 费用到全局

```
if result.liquidity > 0:
    step.feeGrowthGlobalX128 += step.feeAmount × Q128 / result.liquidity
    // UnsafeMath.simpleMulDiv（分子不溢出 uint256，无需 512 位精度）
```

### Step 6：处理 Tick 穿越

```
if result.sqrtPriceX96 == step.sqrtPriceNextX96:  // 到达 tick 边界

    if step.initialized:
        liquidityNet = Pool.crossTick(tickNext, feeGrowthGlobal0, feeGrowthGlobal1)
        // crossTick 内部：
        //   feeGrowthOutside0 = feeGrowthGlobal0 - feeGrowthOutside0  ← 翻转
        //   feeGrowthOutside1 = feeGrowthGlobal1 - feeGrowthOutside1  ← 翻转
        //   return info.liquidityNet

        if zeroForOne: liquidityNet = -liquidityNet  // 向左穿越取反

        result.liquidity = addDelta(result.liquidity, liquidityNet)

    result.tick = zeroForOne ? step.tickNext - 1 : step.tickNext
    //            ↑ 向左 -1 修正（见下方解释）

else if result.sqrtPrice != step.sqrtPriceStart:
    result.tick = TickMath.getTickAtSqrtPrice(result.sqrtPriceX96)
```

> **为什么 zeroForOne 时 tick = tickNext - 1？**
>
> 当价格恰好落在 tick N 的下边界时，活跃流动性区间为 `[N-1, N)`，归属 tick N-1。
> 若不做 -1 修正，`slot0.tick` 与实际活跃区间不一致，会导致 `donate()` 将费用分配到错误的 LP。

---

## 五、循环结束后的状态写回

```solidity
// 一次 sstore 写入价格和 tick（Slot0 打包）
self.slot0 = slot0Start.setTick(result.tick).setSqrtPriceX96(result.sqrtPriceX96);

// 流动性有变化才写（节省 SSTORE）
if self.liquidity != result.liquidity:
    self.liquidity = result.liquidity;

// 只有一侧费用变化
if zeroForOne:
    self.feeGrowthGlobal0X128 = step.feeGrowthGlobalX128;
else:
    self.feeGrowthGlobal1X128 = step.feeGrowthGlobalX128;
```

---

## 六、计算最终用户 Delta

```solidity
// "if currency1 is specified"：zeroForOne 且 exactOutput，或 oneForZero 且 exactInput
if (zeroForOne) != (amountSpecified < 0):
    swapDelta = (amountCalculated, amountSpecified - amountSpecifiedRemaining)
else:
    swapDelta = (amountSpecified - amountSpecifiedRemaining, amountCalculated)
```

| 场景 | delta.amount0 | delta.amount1 |
|---|---|---|
| zeroForOne + exactInput | 实际消耗的 token0（负） | 产生的 token1（正） |
| zeroForOne + exactOutput | 消耗的 token0（负） | 指定的 token1 输出（正） |
| oneForZero + exactInput | 产生的 token0（正） | 实际消耗的 token1（负） |
| oneForZero + exactOutput | 指定的 token0 输出（正） | 消耗的 token1（负） |

---

## 七、数学基础：集中流动性 AMM 公式

Uniswap v4 使用 **恒定乘积 x·y = k** 的变形，引入 `L = √(x·y)`：

```
x = L / sqrtPrice
y = L × sqrtPrice

在价格从 sqrtPA 到 sqrtPB 的一步内：
    Δx = L × (1/sqrtPA - 1/sqrtPB) = L × (sqrtPB - sqrtPA) / (sqrtPA × sqrtPB)
    Δy = L × (sqrtPB - sqrtPA)
```

`sqrtPriceX96` 以 **Q64.96** 固定小数点存储（×2^96），所有 `getAmount0/1Delta` 用 `FullMath.mulDiv` 做 512 位中间精度乘除，避免溢出。

---

## 八、关键设计亮点

| 设计 | 目的 |
|---|---|
| `amountSpecified` 符号区分 exactIn/Out | 单参数双语义，无需额外 bool |
| While 循环跨 tick | 支持任意价格滑动距离，无需预设步数 |
| `tickBitmap` 单 word 内搜索 | 一次 SLOAD 处理 256 个 tick |
| `crossTick` 翻转 `feeGrowthOutside` | O(1) 维护费用分配，无需遍历仓位 |
| `tick = tickNext - 1` 修正 | 保持 `slot0.tick` 与活跃流动性区间一致 |
| 协议费从 total fee 切割 | 先扣协议费，剩余全归 LP，计算简洁 |
| `feeGrowthGlobal` 允许溢出 | uint256 自然回绕，差值语义仍正确 |
| Slot0 打包一次写回 | 减少 SSTORE 次数，节省 gas |
