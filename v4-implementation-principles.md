# Uniswap v4 功能实现原理

---

## 一、Flash Accounting（闪电记账）实现原理

### 核心数据结构

v4 使用 **EIP-1153 瞬态存储（transient storage）** 跟踪每笔操作的余额变化，交易结束后自动清零，无 gas 退款问题。

```
瞬态存储 key = keccak256(currency_address, caller_address)
瞬态存储 value = int256（该调用者对该 currency 的净 delta）
```

### 完整流程

```
用户调用 unlock(data)
    └─ PoolManager:
         1. nonzeroDeltaCount = 0（瞬态）
         2. lock 标志 = UNLOCKED（瞬态）
         3. 回调 msg.sender.unlockCallback(data)

在 unlockCallback 内，用户可调用：
    swap()  →  Pool.swap()  →  accountPoolBalanceDelta()
                                  └─ CurrencyDelta.applyDelta(currency0, caller, amount0)
                                  └─ CurrencyDelta.applyDelta(currency1, caller, -amount1)
                                     若某个 delta 从 0 变非零：nonzeroDeltaCount++
                                     若某个 delta 从非零变 0：nonzeroDeltaCount--

    settle() →  读取 sync() 时保存的余额快照，与当前余额作差 = 实际转入量
                CurrencyDelta.applyDelta(currency, caller, -transferredAmount)
                若 delta 归零：nonzeroDeltaCount--

    take()   →  直接 ERC20.transfer(to, amount)
                CurrencyDelta.applyDelta(currency, caller, +amount)

回调结束后：
    PoolManager 检查 nonzeroDeltaCount == 0，否则 revert CurrencyNotSettled
```

**关键**：`nonzeroDeltaCount` 是瞬态 uint256，追踪"还有多少个未结清的 delta"，不是具体哪个 delta 是多少，逻辑极简。

---

## 二、Hooks 系统实现原理

### 地址编码权限的工作方式

Hook 合约在**构造时**调用 `Hooks.validateHookPermissions()` 验证自己的地址是否符合声明的权限：

```solidity
// Hooks.sol
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;  // 直接位运算，0 gas
}
```

PoolManager 在初始化池子时调用 `Hooks.isValidHookAddress()` 验证：
- delta-return flags 不能在没有对应 action flag 的情况下单独存在
- 若地址非 0，必须至少有一个 flag 或启用动态费率

### Hook 调用的具体实现

以 `beforeSwap` 为例：

```solidity
// Hooks.sol - beforeSwap()
function beforeSwap(...) internal returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride) {
    amountToSwap = params.amountSpecified;

    // noSelfCall modifier：若 hook 自己触发的 swap，跳过回调（防止无限递归）
    if (msg.sender == address(self)) return (...);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        // 低级 call，不走 ABI decoder（节省 gas）
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, ...));

        // 返回值必须是 96 字节：bytes4 selector + int256 delta + uint24 fee
        if (result.length != 96) revert InvalidHookResponse();

        // 动态费率池才解析 fee override
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        // 若有 RETURNS_DELTA 权限，解析 hook 对 amountSpecified 的调整
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();
            amountToSwap += hookDeltaSpecified;  // 调整实际 swap 量
            // 若调整后方向反转（exactIn → exactOut）则 revert
        }
    }
}
```

`callHook` 用内联 assembly 实现，失败时用 ERC-7751 包装错误冒泡：

```solidity
assembly ('memory-safe') {
    success := call(gas(), self, 0, add(data, 0x20), mload(data), 0, 0)
}
if (!success) CustomRevert.bubbleUpAndRevertWith(address(self), bytes4(data), HookCallFailed.selector);
```

---

## 三、集中流动性 Swap 算法原理

### 单步计算（SwapMath.computeSwapStep）

每一步 while 循环都在一个价格区间内计算：

```
给定：sqrtPriceCurrent, sqrtPriceTarget, liquidity, amountRemaining, feePips

exactInput（amountRemaining < 0）:
    max_amountIn = getAmount0/1Delta(sqrtPriceCurrent, sqrtPriceTarget, liquidity, roundUp)
    if |amountRemaining| >= max_amountIn + fee:
        # 能到达 target price
        sqrtPriceNext = sqrtPriceTarget
        amountIn = max_amountIn
    else:
        # 在到达 target 前耗尽输入
        sqrtPriceNext = getNextSqrtPriceFromInput(...)
        amountIn = |amountRemaining| × (1 - feePips/1e6)

feeAmount = amountIn × feePips / (1e6 - feePips)  # 从输入中额外收取
amountOut = getAmount0/1Delta(sqrtPriceCurrent, sqrtPriceNext, liquidity)
```

### Tick 穿越（crossTick）

当价格到达一个已初始化的 tick 边界时：

```solidity
function crossTick(State storage self, int24 tick, uint256 feeGrowthGlobal0, uint256 feeGrowthGlobal1)
    internal returns (int128 liquidityNet)
{
    TickInfo storage info = self.ticks[tick];
    // 翻转 feeGrowthOutside（相对概念）
    info.feeGrowthOutside0X128 = feeGrowthGlobal0 - info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 = feeGrowthGlobal1 - info.feeGrowthOutside1X128;
    liquidityNet = info.liquidityNet;  // 穿越后流动性变化量
}
```

**关键设计**：`feeGrowthOutside` 是相对量，初始化时设为 global 值（若当前 tick 在其下方），穿越时取反，这样无论从哪个方向穿越都能用同一个公式算出 `feeGrowthInside`。

### Tick 方向修正（zeroForOne: tick = tickNext - 1）

```solidity
unchecked {
    result.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
}
```

这是 v3/v4 的经典设计：`sqrtPrice` 可以正好处于 tick N 的下边界，但此时活跃流动性属于 tick N-1 区间。若不做 `-1` 修正，`slot0.tick` 与 `sqrtPrice` 对应的 tick 会不一致，影响 `donate()` 的费用分配逻辑。

---

## 四、费用累积原理（feeGrowthGlobal + feeGrowthOutside）

这是整个系统中最精妙的设计：

```
feeGrowthGlobal 是单调递增的全局累积量（per unit liquidity）

feeGrowthOutside[tick] 的含义：
  "该 tick 对应边界外侧（相对于当前价格）的费用累积量"

当当前 tick >= tickLower 时：
  feeGrowthInside = feeGrowthGlobal - lower.feeGrowthOutside - upper.feeGrowthOutside
         ↑ 只有价格在区间内时，这里才有值累积

LP 实际收益 = liquidity × (feeGrowthInside_now - feeGrowthInside_atLastUpdate) / Q128
```

整个系统只需维护：
- 1 个全局 `feeGrowthGlobal`（每步 swap 更新）
- 每个已初始化 tick 各 2 个 `feeGrowthOutside`（穿越时翻转）
- 每个仓位 1 个 `feeGrowthInside_lastUpdated`（操作时快照）

无需遍历，O(1) 计算任意仓位的待收费用。

---

## 五、TickBitmap 快速查找原理

```
tickBitmap: mapping(int16 wordPos => uint256 word)

tick 的坐标拆分：
    wordPos = tick >> 8   (高位)
    bitPos  = tick & 0xFF (低8位，0~255)

查找下一个初始化的 tick（向左，zeroForOne）：
    mask = (1 << bitPos) - 1 + (1 << bitPos)  // bitPos 及其左边的所有位
    masked = word & mask
    if masked != 0:
        找到 masked 中最高位 → 对应的 tick 就是下一个已初始化 tick
    else:
        移动到上一个 word（wordPos - 1），取该 word 最高位
```

每个 word 代表 256 个 tick，一次 SLOAD 可处理 256 个 tick 的查找，极大减少 storage 访问。

---

## 六、Slot0 位操作与 updateTick Assembly 优化

`Pool.updateTick()` 中用一次 `sstore` 同时写 `liquidityGross + liquidityNet`：

```solidity
assembly ('memory-safe') {
    sstore(
        info.slot,
        or(
            and(liquidityGrossAfter, 0xffffffffffffffffffffffffffffffff),  // 低128位
            shl(128, liquidityNet)                                          // 高128位
        )
    )
}
```

`liquidityGross`（uint128）+ `liquidityNet`（int128）恰好占满 256 位，一次 `sstore` 写完，比分两次便宜约 5000 gas（省一个 SSTORE 冷写入）。

---

## 七、Native ETH 支持实现

v4 中 `Currency(address(0))` 代表原生 ETH：

```
  sync(ETH)    → 无需 snapshot（ETH 的输入量由 msg.value 决定）
  settle()     → 若 currency == address(0)，用 msg.value 结算
                 若还有 ETH 剩余（Native 值多于 delta），退还给 msg.sender
  take(ETH)    → SafeTransferLib.safeTransferETH(to, amount)
```

需要注意：ERC20 的结算流程需要先 `sync(currency)` 快照，然后实际转账，再 `settle()`。而 ETH 不需要 `sync`，因为合约余额变化量 = `msg.value`，直接用即可。

---

## 总结

这七个机制组合在一起，构成了 v4 既比 v3 **更灵活**（Hooks、动态费率、Salt 仓位）又**更省 gas**（单合约、Flash Accounting、Transient Storage、Slot0 打包）的核心竞争力。

| 机制 | 核心技术 | 解决的问题 |
|---|---|---|
| Flash Accounting | EIP-1153 瞬态存储 + delta 计数 | 多操作一次结算，支持闪电贷 |
| Hooks | 地址位图编码权限 | 无需 fork 即可扩展池子行为 |
| Swap 算法 | 跨 tick 迭代状态机 | 集中流动性精确定价 |
| 费用累积 | feeGrowthOutside 差分翻转 | O(1) 计算任意仓位费用 |
| TickBitmap | 256位位图分层索引 | 快速查找下一个流动性边界 |
| Slot0 打包 | Assembly 位操作 | 单 SLOAD 获取全部价格状态 |
| Native ETH | Currency(0) 特判 | 省去 WETH 封装/解封 |
