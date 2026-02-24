# SqrtPriceMath：`getAmount1Delta` 详解

> 源文件：[`src/libraries/SqrtPriceMath.sol`](../../v4-core/src/libraries/SqrtPriceMath.sol)

---

## 一、数学本质

Uniswap V3/V4 采用**集中流动性**模型，在单个 tick 区间 $[\sqrt{P_A},\ \sqrt{P_B}]$ 内，token1 的数量由以下公式给出：

$$\boxed{\Delta y = L \cdot (\sqrt{P_B} - \sqrt{P_A})}$$

| 符号 | 含义 |
|---|---|
| $L$ | 流动性（`liquidity`，`uint128`） |
| $\sqrt{P_A}$，$\sqrt{P_B}$ | 价格区间两端的价格平方根 |
| $\Delta y$ | token1 的数量（即 `amount1`） |

**直觉推导**：在 CPMM 中 $y = L \cdot \sqrt{P}$，价格从 $P_A$ 移动到 $P_B$ 时 token1 的变化量恰好是 $L \cdot (\sqrt{P_B} - \sqrt{P_A})$。这是一个**线性关系**，计算比 token0 简单得多。

---

## 二、Q64.96 定点数格式

Solidity 没有浮点数，Uniswap 以 **Q64.96** 格式存储 $\sqrt{P}$：

$$\texttt{sqrtPX96} = \sqrt{P} \times 2^{96}$$

- `FixedPoint96.RESOLUTION = 96`
- `FixedPoint96.Q96 = 2^96`

代入公式展开后变为纯整数计算：

$$\Delta y = L \cdot \frac{|\texttt{sqrtPBX96} - \texttt{sqrtPAX96}|}{2^{96}}$$

---

## 三、代码逐行解析（`uint` 版本）

```solidity
// src/libraries/SqrtPriceMath.sol  Line 234
function getAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity,
    bool    roundUp
) internal pure returns (uint256 amount1) {

    // ① 计算 |sqrtPriceBX96 - sqrtPriceAX96|，无分支绝对差
    uint256 numerator  = absDiff(sqrtPriceAX96, sqrtPriceBX96);

    // ② 分母 = 2^96，用于消去 Q64.96 缩放因子
    uint256 denominator = FixedPoint96.Q96;   // = 1 << 96

    // ③ 提升到 uint256，防止后续乘法溢出
    uint256 _liquidity = uint256(liquidity);

    // ④ 核心计算：amount1 = L * |Δ√P| / 2^96
    //    FullMath.mulDiv 内部使用 512 位中间结果，保证精度
    amount1 = FullMath.mulDiv(_liquidity, numerator, denominator);

    // ⑤ 若有余数且 roundUp=true，则结果 +1（向上取整）
    assembly ("memory-safe") {
        amount1 := add(
            amount1,
            and(gt(mulmod(_liquidity, numerator, denominator), 0), roundUp)
        )
    }
}
```

| 步骤 | 表达式 | 数学含义 |
|:---:|---|---|
| ① | `absDiff(sqrtPA, sqrtPB)` | $\|\Delta\sqrt{P}\| \times 2^{96}$（Q96 格式） |
| ② | `Q96 = 2^96` | 分母，消去 Q96 缩放 |
| ③ | `uint256(liquidity)` | 防止 `uint128 × uint160` 溢出 |
| ④ | `mulDiv(L, Δ√P, 2^96)` | 精确 512 位中间乘法，还原真实 token1 数量 |
| ⑤ | `mulmod(...) > 0 && roundUp` | 余数非零 + 向上取整 → 结果 +1 |

---

## 四、`absDiff` —— 无分支绝对值

```solidity
// Line 214
function absDiff(uint160 a, uint160 b) internal pure returns (uint256 res) {
    assembly ("memory-safe") {
        let diff := sub(a, b)
        // a >= b → mask = 0x000...；a < b → mask = 0xFFF...（算术右移 255 位）
        let mask := sar(255, diff)
        // a >= b: res = 0 XOR (0 + diff)     = a - b
        // a < b:  res = mask XOR (mask + diff) = b - a
        res := xor(mask, add(mask, diff))
    }
}
```

**效果**：无论 `sqrtPriceAX96` 和 `sqrtPriceBX96` 哪个更大，都能正确计算绝对差，调用方无需手动排序。

---

## 五、为什么使用 `FullMath.mulDiv`？

直接写 `liquidity * numerator / denominator` 会溢出：

- `liquidity`：最大 $2^{128} - 1$
- `numerator`：最大 $2^{160} - 1$
- 乘积可达 $2^{288}$，**超过** `uint256`（$2^{256}$）

`FullMath.mulDiv(a, b, c)` 内部用两个 `uint256` 拼接出 **512 位**中间结果，再精确除以 `c`，无精度损失。

> **注释也明确**："Cannot overflow because `type(uint128).max * type(uint160).max >> 96 < (1 << 192)`."  
> 即在 ÷2^96 之后结果必然小于 2^192，安全放入 `uint256`。

---

## 六、`roundUp` 的两种取整语义

| `roundUp` | 数学等价 | 使用场景 |
|:---:|---|---|
| `false` | $\lfloor L \cdot \Delta\sqrt{P} / 2^{96} \rfloor$（向下截断） | 用户**卖出** token0，计算可得到多少 token1，保守估计 |
| `true` | $\lceil L \cdot \Delta\sqrt{P} / 2^{96} \rceil$（向上进位） | 用户**买入** token0，池子多收 1 wei，防止亏损 |

向上取整的汇编实现：
```solidity
// mulmod(L, Δ√P, 2^96) > 0  ←→  整除有余数
// and(..., roundUp)           ←→  余数存在 && 要求进位
amount1 := add(amount1, and(gt(mulmod(_liquidity, numerator, denominator), 0), roundUp))
```

---

## 七、有符号版本（`int128 liquidity`）

```solidity
// Line 278
function getAmount1Delta(uint160 sqrtPriceAX96, uint160 sqrtPriceBX96, int128 liquidity)
    internal pure returns (int256)
{
    return liquidity < 0
        // 移除流动性：用户「收到」token1 → 正值，向下取整（少收 1 wei）
        ?  getAmount1Delta(..., uint128(-liquidity), false).toInt256()
        // 添加流动性：用户「付出」token1 → 负值，向上取整（多收 1 wei 防损）
        : -getAmount1Delta(..., uint128(liquidity),  true).toInt256();
}
```

| `liquidity` 符号 | 操作 | 返回值符号 | 取整方向 |
|:---:|---|:---:|---|
| 正（`> 0`） | **添加**流动性，用户付出 token1 | **负**（debt） | 向上（池多收） |
| 负（`< 0`） | **移除**流动性，用户收到 token1 | **正**（credit） | 向下（用户少得） |

---

## 八、与 `getAmount0Delta` 的对比

| | `getAmount1Delta` | `getAmount0Delta` |
|---|---|---|
| 数学公式 | $L \cdot (\sqrt{P_B} - \sqrt{P_A})$ | $L \cdot \left(\dfrac{1}{\sqrt{P_A}} - \dfrac{1}{\sqrt{P_B}}\right)$ |
| 与 $\sqrt{P}$ 的关系 | **线性**（直接乘） | **倒数**（需额外除法） |
| 代码复杂度 | 低：一次 `mulDiv` | 高：`numerator1 * numerator2 / sqrtPB / sqrtPA` 两步除 |
| 默认排序需求 | 无（`absDiff` 自动处理） | 需要先确保 `sqrtPA < sqrtPB` |

token1（通常是 USDC 等计价代币）是**线性**计算，因为 $y = L\sqrt{P}$；  
token0（通常是 ETH 等基础资产）是**倒数**计算，因为 $x = L / \sqrt{P}$。

---

## 九、完整数值示例

**场景**：ETH/USDC 池，tick 区间覆盖价格 $[1800, 2200]$ USDC/ETH，流动性 $L = 10^{18}$。

$$\sqrt{1800} \approx 42.43 \implies \texttt{sqrtPAX96} = 42.43 \times 2^{96}$$
$$\sqrt{2200} \approx 46.90 \implies \texttt{sqrtPBX96} = 46.90 \times 2^{96}$$

$$\Delta\sqrt{P}\text{（Q96）} = (46.90 - 42.43) \times 2^{96} = 4.47 \times 2^{96}$$

$$\Delta y = 10^{18} \times \frac{4.47 \times 2^{96}}{2^{96}} = 4.47 \times 10^{18} \text{ wei USDC}$$

即该流动性仓位在此价格区间内需要约 **4.47 USDC**（$4.47 \times 10^{18}$ wei）的 token1 覆盖。

---

## 十、调用链总结

```
Pool.modifyLiquidity() 或 Pool.swap()
    └─ SqrtPriceMath.getAmount1Delta(sqrtPA, sqrtPB, int128 liquidity)  ← 有符号版
            └─ getAmount1Delta(sqrtPA, sqrtPB, uint128 liquidity, roundUp) ← 无符号版
                    ├─ absDiff(sqrtPA, sqrtPB)          → Δ√P（Q96 格式）
                    ├─ FullMath.mulDiv(L, Δ√P, 2^96)    → 主结果（截断）
                    └─ mulmod 汇编                       → 余数进位（roundUp）
```
