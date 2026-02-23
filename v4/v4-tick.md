# Tick 详解

## 一、Tick 是什么

**Tick 是价格空间的离散化刻度标记**，本质是一个 `int24` 整数，代表一个具体的价格点。

### Tick 与价格的关系

```
price(tick) = 1.0001^tick

相邻两个 tick 之间，价格恰好相差 0.01%（1个基点）
```

| tick | 价格 |
|---|---|
| 0 | 1.000000 |
| 1 | 1.000100 |
| 100 | ≈ 1.010050 |
| -100 | ≈ 0.990050 |

**范围**：`MIN_TICK = -887272`，`MAX_TICK = 887272`

---

## 二、Tick 的三种角色

### 角色 1：表示当前价格（slot0.tick）

`slot0.tick` 始终满足：

```
price(tick) ≤ currentPrice < price(tick + 1)
```

价格随 swap 移动时，tick 随之变化。

---

### 角色 2：LP 仓位的价格边界（tickLower / tickUpper）

LP 提供集中流动性时必须指定区间：

```solidity
int24 tickLower;   // 区间下界
int24 tickUpper;   // 区间上界
```

> 只有当 `tickLower ≤ slot0.tick < tickUpper` 时，仓位才是"活跃的"（参与交易、赚取手续费）

---

### 角色 3：Swap 中的流动性切换点（crossTick）

Swap 时价格沿 tick 轴移动，**每穿越一个已初始化的 tick，活跃流动性 `L` 就会变化**：

```
价格向右移动：
    tick:  ─┼──────────┼──────────┼──
              lower(+L)  upper(-L)

穿越 lower tick → pool.liquidity += liquidityNet
穿越 upper tick → pool.liquidity += liquidityNet（负值，减少）
```

---

## 三、tickSpacing：不是每个 tick 都可用

LP 只能把 `tickSpacing` 整数倍的 tick 设为边界：

| tickSpacing | 适用场景 | 精度 |
|---|---|---|
| 1 | 稳定币（价格几乎不变） | 最高 |
| 10 | 中等波动资产 | 中等 |
| 60 | 高波动资产 | 较低 |

tickSpacing 越大，Gas 越省，但 LP 价格精度越低。

---

## 四、已初始化 vs 未初始化的 tick

只有当 LP 将 tick 设为边界时才被**初始化**（写入 `TickInfo`，`tickBitmap` 对应 bit 设为 1）：

```
tickBitmap:   0（未初始化）    1（已初始化）
─────────────────────────────────────────
tick:        -900  -600  -300   0   300
bit:           0     1     0    1    1
```

Swap 时用 `tickBitmap.nextInitializedTickWithinOneWord()` 位运算快速跳过未初始化的 tick。

---

## 五、TickInfo 结构体

每个已初始化 tick 存储 `TickInfo`（512 位 = 2 个 storage slot）：

```solidity
struct TickInfo {
    uint128 liquidityGross;       // 引用该 tick 的所有仓位的流动性总量
    int128  liquidityNet;         // 从左向右穿越时活跃流动性的净变化量
    uint256 feeGrowthOutside0X128; // 外侧 token0 累计费用
    uint256 feeGrowthOutside1X128; // 外侧 token1 累计费用
}
```

### liquidityGross

- `== 0`：没有仓位引用，从 tickBitmap 中移除（flip）
- `> 0`：至少有一个仓位在引用，保持已初始化

### liquidityNet

```
lower tick: liquidityNet = +liquidityDelta  （进入区间，流动性增加）
upper tick: liquidityNet = -liquidityDelta  （离开区间，流动性减少）
```

**存储优化**：前两个字段恰好 256 位，用一次 `sstore` 原子写入：

```solidity
assembly ("memory-safe") {
    sstore(info.slot, or(
        and(liquidityGrossAfter, 0xffffffffffffffffffffffffffffffff), // 低128位
        shl(128, liquidityNet)                                         // 高128位
    ))
}
```

### feeGrowthOutside

**"相对于当前价格，tick 另一侧（外侧）的累计费用"**

- tick 首次初始化时：若当前价格在 tick 右侧，`feeGrowthOutside = feeGrowthGlobal`
- tick 被穿越时（crossTick）：翻转内外 `feeGrowthOutside = feeGrowthGlobal - feeGrowthOutside`

计算任意区间内的费用（O(1)）：

```
价格在区间内：feeGrowthInside = feeGrowthGlobal - lower.outside - upper.outside
价格在区间左：feeGrowthInside = lower.outside - upper.outside
价格在区间右：feeGrowthInside = upper.outside - lower.outside
```

---

## 六、一句话总结

> **Tick 是 Uniswap 对连续价格空间的离散化编号，相邻 tick 价格差 0.01%。它同时承担三个角色：记录当前价格位置、定义 LP 仓位的价格边界、在 Swap 时作为流动性切换的触发点。**
