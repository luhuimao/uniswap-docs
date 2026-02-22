# Uniswap v4 安全性与 Gas 优化借鉴

---

## 🔒 安全性设计

### 1. Flash Accounting 的"先操作后结算"强制校验

```solidity
// 回调结束后统一校验，不依赖业务逻辑各自检查
if (nonzeroDeltaCount != 0) revert CurrencyNotSettled();
```

**借鉴点**：将"资金守恒"验证从分散的业务代码中抽离，放在最外层统一兜底检查。即使内部逻辑有疏漏，出口处的不变量检查能保住最后一道防线。

---

### 2. Hook 权限不可伪造（地址即权限）

Hook 的可执行权限由**部署地址的二进制位**决定，调用时直接位运算：

```solidity
return uint160(address(self)) & flag != 0;
```

**借鉴点**：权限与地址强绑定，无法在运行时动态修改，彻底消除权限提升攻击面。比 `mapping(address => bool)` 的权限表更安全（不存在管理员误操作或被攻击的风险）。

---

### 3. `noSelfCall` 防止 Hook 无限递归

```solidity
modifier noSelfCall(IHooks self) {
    if (msg.sender != address(self)) { _; }
}
```

**借鉴点**：在可扩展系统中，若 Hook/Plugin 本身也能触发宿主行为，必须用简单的 `msg.sender` 检查阻断递归调用，防止重入或无限循环。

---

### 4. Hook 调用失败的错误冒泡（ERC-7751）

```solidity
if (!success) CustomRevert.bubbleUpAndRevertWith(
    address(self), bytes4(data), HookCallFailed.selector
);
```

**借鉴点**：外部调用失败时，用结构化方式保留原始错误上下文，方便调试。比 `require(success, "hook failed")` 提供的信息更丰富。

---

### 5. 精确的错误类型（Custom Errors + 携带参数）

```solidity
error TicksMisordered(int24 tickLower, int24 tickUpper);
error PriceLimitAlreadyExceeded(uint160 sqrtPriceCurrent, uint160 sqrtPriceLimit);

TicksMisordered.selector.revertWith(tickLower, tickUpper);
```

**借鉴点**：自定义错误携带上下文参数，既省 gas（比字符串便宜），又让链上和链下调试更精确。

---

### 6. `feeGrowthGlobal` 溢出的有意设计

```solidity
unchecked {
    feesOwed0 = FullMath.mulDiv(
        feeGrowthInside0X128 - self.feeGrowthInside0LastX128,  // 可回绕，结果仍正确
        liquidity,
        FixedPoint128.Q128
    );
}
```

**借鉴点**：对于单调递增的累积量，允许自然溢出（模2^256），差值计算在 unchecked 下仍语义正确。理解这一点能避免对溢出过度防御导致逻辑错误。

---

### 7. `clear()` 的精确金额要求

```solidity
// 放弃粉尘时必须传入精确金额，不允许"清零所有"
function clear(Currency currency, uint256 amount) external;
// 若 amount 与实际 delta 不匹配：revert MustClearExactPositiveDelta
```

**借鉴点**：强制调用者明确知道自己在放弃多少，防止因误操作或被前置抢先交易（frontrun）而无意中清除了非粉尘金额。

---

## ⚡ Gas 优化技巧

### 1. EIP-1153 瞬态存储（Transient Storage）

```solidity
// 用 tstore/tload 替代 sstore/sload 存储跨调用的临时状态
// tstore: ~100 gas vs sstore(冷): ~20000 gas
assembly { tstore(slot, value) }
```

**借鉴点**：凡是"交易内有效、交易结束后不需要"的状态（锁、delta 计数、余额快照），都应使用瞬态存储。

---

### 2. Slot0 全状态打包在一个 bytes32

```
256 bits: [24空][lpFee:24][protocolFee:24][tick:24][sqrtPriceX96:160]
```

```solidity
// 一次 SLOAD 读出全部核心状态
Slot0 slot0Start = self.slot0;
(int24 tick, uint160 sqrtPriceX96) = (slot0Start.tick(), slot0Start.sqrtPriceX96());
```

**借鉴点**：频繁一起读取的字段应打包到同一个 storage slot。每减少一次 SLOAD，冷读节省约 2100 gas，热读节省 100 gas。

---

### 3. 单次 `sstore` 写两个相邻字段

```solidity
// liquidityGross（uint128）+ liquidityNet（int128）= 256位，一次写完
assembly ('memory-safe') {
    sstore(info.slot, or(
        and(liquidityGrossAfter, 0xffffffffffffffffffffffffffffffff),
        shl(128, liquidityNet)
    ))
}
```

**借鉴点**：若两个字段加起来恰好 ≤ 256 位，用 assembly 手动打包写入，节省 SSTORE 次数。

---

### 4. BalanceDelta：双代币 delta 打包为 int256

```solidity
type BalanceDelta is int256;
// 高128位 = amount0，低128位 = amount1
// 加减法全部在 assembly 中完成，无解包开销
```

**借鉴点**：在需要同时传递/操作两个相关数值的场景，自定义值类型（User-Defined Value Types）+ 运算符重载是优雅且省 gas 的方式。

---

### 5. TickBitmap：用位图索引稀疏数据

```solidity
mapping(int16 wordPos => uint256 word) tickBitmap;
// 一次 SLOAD 覆盖 256 个 tick，用位操作找最近的初始化 tick
```

**借鉴点**：对于稀疏的布尔集合（如"哪些元素已初始化"），位图比 `mapping(key => bool)` 更高效，既省存储又减少查找时的 SLOAD 次数。

---

### 6. `IExtsload`：外部合约直读 storage

```solidity
// 无需为每个字段写 view 函数，外部合约直接指定 slot 读取
bytes32 data = manager.extsload(stateSlot);
bytes32[] memory data = manager.extsload(slot, 3);  // 批量读取
```

**借鉴点**：复杂合约可暴露 `extsload` 接口，让外部工具/协议自行读取所需数据，避免维护大量 view 函数，同时支持批量读取节省调用开销。

---

### 7. `unchecked` 的精准使用

v4 中 `unchecked` 不是随便用，每处都有明确注释说明为何安全：

```solidity
unchecked {
    // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
    amountSpecifiedRemaining += (step.amountIn + step.feeAmount).toInt256();
}
```

**借鉴点**：`unchecked` 应配合注释说明溢出不可能发生的数学原因，而不是盲目使用。

---

### 8. Assembly `memory-safe` 标注

```solidity
assembly ('memory-safe') {
    // 明确声明只访问已分配的内存区域
    // 允许编译器进行更好的优化
}
```

**借鉴点**：当 assembly 块确实不会破坏 Solidity 的内存模型时，加上 `'memory-safe'` 标注，编译器可对周围的 Solidity 代码做更多优化。

---

## 总结对比表

| 技术 | 安全 | Gas | 适用场景 |
|---|:---:|:---:|---|
| 出口统一不变量校验 | ✅ | | 多步骤复合操作 |
| 地址编码权限 | ✅ | ✅ | 插件/扩展系统 |
| `noSelfCall` 防递归 | ✅ | | Hook/回调系统 |
| 结构化错误冒泡 | ✅ | | 外部 call 调用链 |
| Custom Error + 参数 | ✅ | ✅ | 任何合约 |
| 瞬态存储 (EIP-1153) | | ✅ | 跨调用临时状态 |
| Slot 打包 / 手动位操作 | | ✅ | 频繁读写的状态 |
| UDT + 运算符重载 | | ✅ | 相关联的数值对 |
| 位图索引稀疏集合 | | ✅ | 稀疏布尔状态 |
| `extsload` 接口 | | ✅ | 大型单例合约 |
| `unchecked` + 注释 | | ✅ | 已证明安全的算术 |
| `memory-safe` assembly | | ✅ | 内联 assembly |
