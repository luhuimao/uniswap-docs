# Uniswap v4 深度解析文档

本仓库收录了对 Uniswap v4-core 协议的技术分析文档，涵盖架构总览、功能实现原理、安全与 Gas 优化等维度。

---

## 📄 文档目录

### [v4-flowcharts.md](./v4-flowcharts.md)
**Uniswap v4 核心流程图**

用 Mermaid 图直观呈现 v4 的执行路径，包括：
- 整体交互时序图（Flash Accounting unlock ↔ callback ↔ settle）
- Swap 完整流程（beforeSwap Hook → 跨 Tick 循环 → afterSwap Hook）
- 添加/移除流动性流程（tick 更新 → 费用快照 → delta 计算）
- Hook 权限位图（14 个标志位说明）
- ERC20 / ETH / ERC6909 费用结算流程对比
- Pool.State 存储结构 ER 图

---

### [v4-liquidity-management.md](./v4-liquidity-management.md)
**Uniswap v4 流动性池管理详解**

深入讲解 v4 如何管理流动性池的每个环节，包括：
- PoolKey 五元组标识与 PoolId 生成
- Pool.State 完整状态结构（Slot0 打包、ticks、positions、tickBitmap）
- 添加流动性的分步流程（tick 更新 → 费用快照 → Position 更新 → delta 计算）
- 移除流动性与 tick 数据清理
- Swap 时跨 tick 的流动性切换（crossTick / liquidityNet）
- Position Salt 机制（同 owner 多仓位）
- 费用结算的 O(1) 差分原理

---

### [uniswap_v4_core_analysis.md](./uniswap_v4_core_analysis.md)

**Uniswap v4-core 代码深度分析**

对 v4-core 整体代码库的结构性解读，包括：
- 目录结构总览（interfaces / libraries / types）
- Flash Accounting 闪电记账架构
- Hooks 系统与地址编码权限机制
- Pool 状态管理与 Slot0 极致打包
- Swap 跨 Tick 迭代算法
- 费率体系（静态费率 / 动态费率 / 协议费率）
- ERC-6909、IExtsload、原生 ETH 支持等新特性
- v3 vs v4 对比总结

---

### [v4-implementation-principles.md](./v4-implementation-principles.md)
**Uniswap v4 功能实现原理**

深入每个核心功能的具体实现细节，包括：
- Flash Accounting 完整执行流程与瞬态存储的使用
- Hooks 的调用链路、delta 机制与动态费率覆盖实现
- 集中流动性 Swap 算法（单步计算 + Tick 穿越 + 方向修正）
- 费用累积的差分翻转原理（O(1) 计算任意仓位收益）
- TickBitmap 快速查找的位图索引实现
- Assembly 级别的 Slot0 打包与 updateTick 单次写入优化
- 原生 ETH 的结算特判逻辑

---

### [v4-security-and-gas-optimization.md](./v4-security-and-gas-optimization.md)
**Uniswap v4 安全性与 Gas 优化借鉴**

从 v4 源码中提炼的工程实践，可直接应用于其他智能合约项目：

**安全性**：
- 出口统一不变量校验（替代分散检查）
- 地址编码权限（不可运行时篡改）
- `noSelfCall` 防递归回调
- ERC-7751 结构化错误冒泡
- Custom Error 携带参数
- 有意溢出（cumulative 差值语义）
- 精确金额校验防止意外清除

**Gas 优化**：
- EIP-1153 瞬态存储替代临时 storage
- Storage slot 紧密打包策略
- 单次 `sstore` 写多个字段
- 用户自定义值类型（UDT）+ 运算符重载
- 位图索引稀疏布尔集合
- `extsload` 接口暴露内部存储
- `unchecked` 精准使用规范
- `memory-safe` assembly 标注

---

## 🗂️ 数据来源

分析基于 Uniswap v4-core 官方源码：

```
contracts/src/briefcase/protocols/v4-core/
├── interfaces/   # IPoolManager、IHooks 等核心接口
├── libraries/    # Pool、Hooks、SwapMath 等核心逻辑库
└── types/        # PoolKey、Slot0、BalanceDelta 等自定义类型
```

