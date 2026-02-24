# 钩子（Hooks）

在所有新特性中，**钩子**是 Uniswap V4 最重要的补充。

在 Uniswap V3 和其他 DEX 中，用户只能与协议内置的预定义功能进行交互。然而，Uniswap V4 允许池创建者通过钩子自定义他们的池，添加额外逻辑。这样一来，池可以提供适应特定需求的新功能。

根据 Uniswap 的 V4 博客文章，钩子可以使得以下功能成为可能：

- **TWAMM**（时间加权自动做市商）
- 根据波动率动态调整费用
- 交易所内限价单
- 将超出范围的流动性存入借贷协议
- 自定义链上预言机
- 自动复利 LP 费用
- 将 MEV 利润重新分配给 LP

但钩子如何实现这些功能的？它们能够带来什么其他的用例？开发者们又在构建什么？

接下来，我们将详细探讨钩子，分析它们在 Uniswap V4 中的影响和潜力。

---

## 深入探讨 Uniswap V4 钩子（Hook）

### 钩子的整体行为

在编程中，**钩子**指的是在软件组件之间拦截函数调用、消息或事件，以修改或扩展其功能。负责处理这些拦截消息或函数调用的组件称为钩子。这一模式在调试和扩展软件能力时广泛使用。

### 钩子的十个回调函数

Uniswap V4 利用这一编程模式，使得构建者能够扩展 DEX 的功能。在 Uniswap V4 中，共定义了 **10 个钩子回调函数**，覆盖池的全部核心操作：

| 函数名称 | 触发时机 |
|---|---|
| `beforeInitialize` | 池初始化之前 |
| `afterInitialize` | 池初始化之后 |
| `beforeAddLiquidity` | 添加流动性之前 |
| `afterAddLiquidity` | 添加流动性之后（可返回 BalanceDelta） |
| `beforeRemoveLiquidity` | 移除流动性之前 |
| `afterRemoveLiquidity` | 移除流动性之后（可返回 BalanceDelta） |
| `beforeSwap` | 交换执行之前（可返回 BeforeSwapDelta 和费用覆写） |
| `afterSwap` | 交换执行之后（可返回 int128 delta） |
| `beforeDonate` | 捐赠之前 |
| `afterDonate` | 捐赠之后 |

如功能名称所示，Uniswap V4 允许在特定操作发生前或后进行钩入，从而根据与该操作相关的数据实现自定义操作。例如，`beforeSwap` 函数在交换执行前被调用，允许钩子开发者在 `PoolManager` 处理交换前实施自定义逻辑。

一个钩子可以实现单个函数或多个函数，意味着开发者几乎可以将他们希望的逻辑集成到 Uniswap V4 的每个动作中。

---

### 钩子的结构

在 Uniswap V4 中，钩子是与流动性池**独立的单独智能合约**。在部署池时，部署者必须指定与池关联的钩子的地址。一旦设置钩子，就无法更改。然而，同一个钩子可以在多个池中共享。

重要的是，即使两个池共享相同的代币对（例如，USDC-ETH）和相同的费用层级，如果它们使用不同的钩子，它们仍被视为不同的池。在 Uniswap V3 中，基于费用层级，代币对只能存在于六个不同的池中。而相比之下，Uniswap V4 允许多个不同功能的 USDC-ETH 池的版本，例如一个集成预言机的池、一个支持限价单的池或一个提供动态费用的池。

---

## 钩子接口：`IHooks.sol`

> 源文件：[`src/interfaces/IHooks.sol`](../../v4-core/src/interfaces/IHooks.sol)

所有钩子合约须实现 `IHooks` 接口。以下是完整的函数签名，展示了每个回调的 **输入参数** 和 **返回值**：

```solidity
interface IHooks {

    // ── 池初始化 ──────────────────────────────────────────────────────────
    function beforeInitialize(address sender, PoolKey calldata key, uint160 sqrtPriceX96)
        external returns (bytes4);

    function afterInitialize(address sender, PoolKey calldata key, uint160 sqrtPriceX96, int24 tick)
        external returns (bytes4);

    // ── 流动性管理 ───────────────────────────────────────────────────────
    function beforeAddLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4);

    // afterAddLiquidity 可主动返回 BalanceDelta，影响调用方的代币结算
    function afterAddLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        BalanceDelta delta,       // 主结算 delta
        BalanceDelta feesAccrued, // 本次已累积手续费
        bytes calldata hookData
    ) external returns (bytes4, BalanceDelta);

    function beforeRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4);

    function afterRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        BalanceDelta delta,
        BalanceDelta feesAccrued,
        bytes calldata hookData
    ) external returns (bytes4, BalanceDelta);

    // ── 交换 ─────────────────────────────────────────────────────────────
    // beforeSwap 返回三个值：
    //   bytes4           - 函数选择器（验证用）
    //   BeforeSwapDelta  - 钩子对 specified/unspecified 金额的主动调整
    //   uint24           - lpFeeOverride（仅动态费用池生效，需带 OVERRIDE_FEE_FLAG）
    function beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4, BeforeSwapDelta, uint24);

    // afterSwap 返回 int128，表示钩子对 unspecified 代币的额外 delta
    function afterSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata hookData
    ) external returns (bytes4, int128);

    // ── 捐赠 ─────────────────────────────────────────────────────────────
    function beforeDonate(
        address sender, PoolKey calldata key,
        uint256 amount0, uint256 amount1,
        bytes calldata hookData
    ) external returns (bytes4);

    function afterDonate(
        address sender, PoolKey calldata key,
        uint256 amount0, uint256 amount1,
        bytes calldata hookData
    ) external returns (bytes4);
}
```

> **注意**：每个钩子函数 **必须** 在成功时返回自身的函数选择器（`bytes4`），`PoolManager` 会校验该值。若返回的选择器不匹配，将抛出 `InvalidHookResponse()` 错误并回滚交易。

---

## 钩子权限系统：`Hooks.sol`

> 源文件：[`src/libraries/Hooks.sol`](../../v4-core/src/libraries/Hooks.sol)

`Hooks` 库是整个钩子系统的核心引擎，负责**权限检查**、**调用分发**和**返回值处理**。

### 权限标志常量

钩子地址的低 14 位（bit 0–13）作为权限标志，每一位对应一个钩子函数：

```solidity
library Hooks {
    uint160 internal constant ALL_HOOK_MASK = uint160((1 << 14) - 1);

    // 初始化
    uint160 internal constant BEFORE_INITIALIZE_FLAG         = 1 << 13;
    uint160 internal constant AFTER_INITIALIZE_FLAG          = 1 << 12;

    // 流动性
    uint160 internal constant BEFORE_ADD_LIQUIDITY_FLAG      = 1 << 11;
    uint160 internal constant AFTER_ADD_LIQUIDITY_FLAG       = 1 << 10;
    uint160 internal constant BEFORE_REMOVE_LIQUIDITY_FLAG   = 1 << 9;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_FLAG    = 1 << 8;

    // 交换
    uint160 internal constant BEFORE_SWAP_FLAG               = 1 << 7;
    uint160 internal constant AFTER_SWAP_FLAG                = 1 << 6;

    // 捐赠
    uint160 internal constant BEFORE_DONATE_FLAG             = 1 << 5;
    uint160 internal constant AFTER_DONATE_FLAG              = 1 << 4;

    // 返回 Delta 标志（需配合对应操作标志共同启用）
    uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG            = 1 << 3;
    uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG             = 1 << 2;
    uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG    = 1 << 1;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;
}
```

对应的位映射总结如下（bit 从高到低）：

| Bit | 值（hex）| 对应标志 |
|:---:|:---:|---|
| 13 | `0x2000` | `BEFORE_INITIALIZE` |
| 12 | `0x1000` | `AFTER_INITIALIZE` |
| 11 | `0x0800` | `BEFORE_ADD_LIQUIDITY` |
| 10 | `0x0400` | `AFTER_ADD_LIQUIDITY` |
| 9 | `0x0200` | `BEFORE_REMOVE_LIQUIDITY` |
| 8 | `0x0100` | `AFTER_REMOVE_LIQUIDITY` |
| 7 | `0x0080` | `BEFORE_SWAP` |
| 6 | `0x0040` | `AFTER_SWAP` |
| 5 | `0x0020` | `BEFORE_DONATE` |
| 4 | `0x0010` | `AFTER_DONATE` |
| 3 | `0x0008` | `BEFORE_SWAP_RETURNS_DELTA` |
| 2 | `0x0004` | `AFTER_SWAP_RETURNS_DELTA` |
| 1 | `0x0002` | `AFTER_ADD_LIQUIDITY_RETURNS_DELTA` |
| 0 | `0x0001` | `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA` |

**示例**：地址 `0x4f....00C0` 的低 16 位为 `0x00C0`，二进制为 `0000 0000 1100 0000`：
- bit 7 = 1 → `BEFORE_SWAP` ✓
- bit 6 = 1 → `AFTER_SWAP` ✓
- 其余位 = 0 → 其他钩子函数均未实现

### 权限检查辅助函数

```solidity
// 检查钩子地址是否带有某个权限标志
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```

### 地址有效性验证

`isValidHookAddress` 函数在池初始化时被调用，确保钩子地址满足以下规则：

```solidity
function isValidHookAddress(IHooks self, uint24 fee) internal pure returns (bool) {
    // delta 返回标志只能在对应操作标志也被设置时才有效
    if (!self.hasPermission(BEFORE_SWAP_FLAG) && self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_SWAP_FLAG) && self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_ADD_LIQUIDITY_FLAG) && self.hasPermission(AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG)) return false;
    if (!self.hasPermission(AFTER_REMOVE_LIQUIDITY_FLAG) && self.hasPermission(AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG)) return false;

    // 无钩子合约时fee不能是动态费用；有钩子合约时必须至少有一个标志或动态费用
    return address(self) == address(0)
        ? !fee.isDynamicFee()
        : (uint160(address(self)) & ALL_HOOK_MASK > 0 || fee.isDynamicFee());
}
```

> **关键规则**："返回 delta" 的标志（bit 0–3）必须配合对应的操作标志同时设置，否则合约部署后将无法通过验证。例如，设置了 `BEFORE_SWAP_RETURNS_DELTA_FLAG` 的地址，必须同时设置 `BEFORE_SWAP_FLAG`。

### 构造函数中验证权限（推荐模式）

Uniswap 提供了 `validateHookPermissions` 工具函数，供钩子合约在 **constructor** 中验证自身地址的权限位与期望的 `Permissions` 结构体是否一致：

```solidity
struct Permissions {
    bool beforeInitialize;
    bool afterInitialize;
    bool beforeAddLiquidity;
    bool afterAddLiquidity;
    bool beforeRemoveLiquidity;
    bool afterRemoveLiquidity;
    bool beforeSwap;
    bool afterSwap;
    bool beforeDonate;
    bool afterDonate;
    bool beforeSwapReturnDelta;
    bool afterSwapReturnDelta;
    bool afterAddLiquidityReturnDelta;
    bool afterRemoveLiquidityReturnDelta;
}

function validateHookPermissions(IHooks self, Permissions memory permissions) internal pure {
    if (
        permissions.beforeSwap != self.hasPermission(BEFORE_SWAP_FLAG)
        || permissions.afterSwap != self.hasPermission(AFTER_SWAP_FLAG)
        // ... 其他权限位校验 ...
    ) {
        HookAddressNotValid.selector.revertWith(address(self));
    }
}
```

**推荐用法**（在钩子 constructor 中）：
```solidity
constructor(IPoolManager _manager) {
    // 验证钩子地址的实际权限位与声明的 Permissions 一致
    IHooks(address(this)).validateHookPermissions(
        Hooks.Permissions({
            beforeSwap: true,
            afterSwap: true,
            beforeSwapReturnDelta: true,
            // 其他字段设为 false ...
        })
    );
}
```

---

## 钩子在底层是如何工作的

### 钩子的核心逻辑

要理解钩子的内部工作原理，必需检查其功能结构。在 Uniswap V4 中，为每个钩子函数定义了特定的输入和输出数据格式。本节探讨这些函数的结构，详细说明钩子从 Uniswap V4 合约接收的数据以及它们可以对这些数据执行的操作。

让我们以 `beforeSwap` / `afterSwap` 为例。

**`beforeSwap` 函数**在 `PoolManager` 执行交换逻辑之前被触发。它从 `PoolManager` 接收五个关键参数：

| 参数 | 描述 |
|---|---|
| `sender` | 发起交换的地址（通常是 Router） |
| `key` | 池的标识符（包含配置信息） |
| `params` | 交换参数（金额、方向、滑点限制等） |
| `hookData` | 传递给钩子的自定义数据 |

这意味着每次发生交换时，钩子可以访问以下关键数据：

- 谁发起了交换
- 希望交换多少
- 交易买入/卖出的是什么代币
- 用户设置的滑点限制

在执行其逻辑后，`beforeSwap` 将以下值返回给 `PoolManager`：

| 返回值 | 描述 |
|---|---|
| `bytes4` | 函数选择器，用于验证钩子调用合法性 |
| `BeforeSwapDelta` | 钩子对交换金额的主动修改 |
| `uint24 lpFeeOverride` | 动态覆写的 LP 费用（可选） |

此时，我们可以看到钩子不仅仅是被动扩展，而是与池的操作进行积极交互的强大机制。

两个关键返回值——`beforeSwapDelta` 和 `lpFeeOverride`——允许钩子动态调整交换流程和费用结构。

---

### `beforeSwapDelta`：对交换的主动影响

`beforeSwapDelta` 值与 Uniswap V4 的**债务结算模型**直接相关。在 V4 中，交换在 Router 和池之间造成临时的债务，这些债务随后将被清偿。类似地，钩子可以与池建立债务关系，从而主动干预交换过程。

例如，钩子可以：

- 获取用户交换的一部分作为费用或返还
- 根据自定义条件修改交易执行路径
- 根据市场状况提供动态的滑点调整

> **注意**：钩子并不要求返回 `beforeSwapDelta` 值。有些钩子可能完全不影响交换机制，而是执行独立、不干扰的操作。

---

### `lpFeeOverride`：动态费用调整

`lpFeeOverride` 与 Uniswap V4 的**动态费用系统**相关。在 Uniswap V4 中，根据市场条件，费用可以动态变化，但只能通过钩子实现。

动态费用实现的两种方法是：

#### 1. 永久性费用更新

钩子可以调用 `updateDynamicLPFee` 函数，永久性调整池的费用，以响应高波动性或低流动性等事件。这允许池：

- 在市场动荡期间提高费用（以补偿 LP 的更高风险）
- 在交易量较低时降低费用，以鼓励更多交换

#### 2. 临时费用调整

通过修改 `lpFeeOverride`，钩子可以仅针对特定用户或交易应用自定义费用，而不是改变池的默认费用。例如：

- 为 NFT 持有者提供折扣费用
- 对套利者收取更高费用
- 为特定交易行为提供定制激励措施

这种灵活性使得 Uniswap V4 能够在**交易级别**实现定制的费用政策，而传统的 AMM 无法做到这一点。

---

### `beforeSwap` 支持的三个关键操作

总之，`beforeSwap` 允许钩子执行三个关键操作：

#### 1. 访问控制

Uniswap V4 钩子可以通过在 `beforeSwap` 函数中验证发送地址，使开发者能够实施访问控制机制。这意味着钩子可以强制执行允许执行交换的用户自定义条件。例如，池可以被设计成只有持有特定 NFT 的用户才能在其中进行交易。

然而，有一点需要注意的是，用户在执行交换时并不直接与 `PoolManager` 交互。相反，他们通常通过 Router 合约进行交互，然后 Router 再与 `PoolManager` 交互。因此，在 `beforeSwap` 中，钩子看到的发送者地址通常是 Router 地址，而不是实际用户地址。

因此，**访问控制逻辑通常是在 Router 级别实现的**，而不是钩子内部。钩子决定哪个 Router 应执行交换，而 Router 负责用户身份验证和权限实施。

#### 2. 干预交换逻辑

Uniswap V4 钩子可以直接在交换执行逻辑中干预。因为钩子从 `PoolManager` 获取所有相关交易细节，所以它们可以动态调整交易执行参数并修改结果。

这一能力促成了各种用例，例如：

- 实施为特定需求量身定制的交换逻辑
- 针对不同类型的池调整执行机制
- 针对不同市场状况优化交换策略

默认情况下，Uniswap V4 使用集中流动性，将价格范围划分为多个涨落区间，每个涨落区间遵循 `x * y = k` CPMM 算法。然而，钩子可以覆盖这一行为，并应用替代的交换逻辑。

例如，钩子可以针对稳定币对实现与 Curve 类似的 **StableSwap 算法**，降低滑点，提高资本效率，尤其是对于大规模交易。

#### 3. 动态费用调整

最后，钩子能够动态调整 LP 费用，为流动性提供者（LP）提供更大的灵活性和潜在收益。

最常见的动态费用模型之一是**基于波动率的费用调整**。在高度波动的情况下，LP 面临非永久性损失（IL）和与再平衡损失（LVR），可能导致经济损失。为补偿这一风险，池可以在高波动时期实施更高的费用，以提升整体 LP 收入并吸引更深的流动性。

> **重要**：钩子无法修改所有池的费用。只有明确启用动态费用的池能被钩子调整费用。这意味着，在固定费用池中交易的用户无需担心不可预测的费用变化。

---

## 钩子如何安装在池中

那么，钩子如何安装在流动性池中？是否可以将任何钩子附加到一个池中？答案是**否定的**。

Uniswap V4 不允许钩子随意附加到任何池。当 `PoolManager` 部署一个池时，必须经过验证程序，以确保指定的钩子满足特定条件。

### 钩子地址编码机制

钩子合约通过将其实现的函数编码到其**合约地址**中与 `PoolManager` 进行通信。这一编码的关键部分是钩子地址的**最后四个十六进制数字（16 位）**。这 16 位表示钩子实现了哪些函数。

例如，假设一个钩子合约的地址是 `0x4f....00C0`。最后四位（`00C0`）是关键部分。`00C0` 是十六进制值，转换成二进制则为：

```
0000 0000 1100 0000
```

这一二进制表示的每一位（`0` 或 `1`）对应一个特定的钩子函数。如果某一位为 `1`，表示钩子实现了该函数；如果为 `0`，则表示该函数未实现。

下表概述了每一位的含义：

| 位（从高到低） | 对应函数 |
|---|---|
| 15 | `beforeInitialize` |
| 14 | `afterInitialize` |
| 13 | `beforeAddLiquidity` |
| 12 | `afterAddLiquidity` |
| 11 | `beforeRemoveLiquidity` |
| 10 | `afterRemoveLiquidity` |
| 9 | `beforeSwap` |
| 8 | `afterSwap` |
| 7 | `beforeDonate` |
| 6 | `afterDonate` |
| 5 | `noOp`（noOp flag for beforeSwap） |
| 4 | `accessLock` |
| 3 | `beforeSwapReturnDelta` |
| 2 | `afterSwapReturnDelta` |
| 1 | `afterAddLiquidityReturnDelta` |
| 0 | `afterRemoveLiquidityReturnDelta` |

在示例 `00C0`（`0000 0000 1100 0000`）中，仅有第 7 位（`beforeSwap`）和第 8 位（`afterRemoveLiquidity`）被设置为 `1`，表示该钩子仅实现这两个函数。

### 为什么将函数编码到地址中？

将钩子函数编码到合约地址本身的原因是**降低 Gas 成本**。

如果 Uniswap 不使用这种方法，`PoolManager` 就无法知道钩子实现了哪些函数。它将不得不尝试调用每一个可能的函数，以确定哪些函数存在。这将导致不必要的函数调用并浪费 Gas。

例如，假设一个钩子仅实现 `beforeSwap`。如果 `PoolManager` 事先不知道这一点，它将试图调用其他函数，比如 `afterSwap`、`beforeAddLiquidity` 等，造成不必要的交易和更高的 Gas 费用。

通过将函数实现直接编码到合约地址中，`PoolManager` 能够很快知道该执行哪些函数。这使得无效的函数调用可以完全跳过，从而降低了 Gas 消耗。

此外，表格中的最后四位（标记为**返回 delta**）指示钩子是否修改了交换或流动性事件的结果。例如，如果一个钩子修改了交换执行逻辑，它应返回一个 `beforeSwapDelta` 值。这一修改的存在通过地址的最后四位表示，从而确保 `PoolManager` 事先知道是否需要对这些变化采取相应的调整。这种设计防止了无用的计算，进一步优化了 Gas 效率。

---

### 使用 CREATE2 进行地址挖矿

由于钩子的功能由其地址决定，开发者必须手动找到符合正确函数配置的合约地址。以太坊的 `CREATE2` 操作码允许开发者通过指定 `salt` 值在预定地址上部署合约。这一技术称为**"地址挖矿"**，使开发者能够搜索一个有效的 `salt`，从而生成具有所需最后四个十六进制数字的地址。

使用 `CREATE2` 生成以太坊合约地址的公式是：

```
address = keccak256(0xff + sender + salt + keccak256(init_code))
```

| 参数 | 说明 |
|---|---|
| `0xff` | 表示 CREATE2 的常量前缀 |
| `sender` | 部署合约的地址 |
| `salt` | 开发者定义的任意值 |
| `init_code` | 正在部署的合约的字节码 |

通过修改 `salt`，开发者可以反复生成不同的地址，直到找到最后四位符合所需函数编码的地址。

### 简化工具

虽然 `CREATE2` 使开发者能够生成正确的钩子地址，但手动搜索正确的 `salt` 可能非常繁琐。此外，测试钩子也变得不便利，因为开发者在每次部署修改过的合约时都必须重复寻找一个新的有效地址。

为简化这一过程，Uniswap 提供了以下工具：

- **[HookMiner](https://github.com/Uniswap/v4-template)**：自动化地址挖矿过程，使开发者能够高效地找到其钩子的有效地址进行测试。
- **[V4 Hook Address Miner](https://uniswaphooks.com/)**（基于 Web 的 UI 工具）：允许开发者：
  - 选择他们希望在钩子中实现的功能
  - 自动生成一个有效地址所需的 `salt`
  - 部署他们的钩子合约，而无需手动挖找有效地址

通过利用这些工具，开发者能够无缝部署格式正确的钩子，从而使 Uniswap V4 钩子生态系统更加开放和便捷。

---

## 潜在考虑因素

到目前为止，我们探讨了 Uniswap V4 钩子的功能及其如何扩大 Uniswap 的能力。然而，这些特性是否引入了任何副作用？

### 每个池的安全性

尽管 Uniswap V4 钩子的安装和输入/输出值存在某些限制，但**对钩子的内部逻辑实施没有限制**。换句话说，恶意开发者可能会设计恶意钩子，窃取交易者或流动性提供者的资金。

例如：

- **DoS 攻击**：恶意开发者可以利用动态费用算法对池实施拒绝服务（DoS）攻击，实际上阻断用户活动。一种可能的攻击场景是，当满足某一价格条件时，将交换费用提高到超过 100%（超过 `MAX_LP_FEE = 1_000_000`，即100%），导致所有交换失败。
- **流动性锁定**：不当的访问控制可能导致池的流动性被冻结或被永久锁定。例如，如果 `beforeInitialize` 函数开放给任何人调用，会导致池中的流动性被永久锁定。

### 用户注意事项

**对于交易者**：使用 Uniswap 官方界面的交易者在交换代币时无需担心恶意钩子。这是因为 Uniswap 集成了 **UniswapX**，一个基于意图的交换执行网络。使用 UniswapX 时，处理与恶意钩子相关的安全风险的责任落在处理用户意图的执行者身上。

**对于流动性提供者（LP）**：LP 需更为谨慎。在 Uniswap V4 中提供流动性时，界面允许用户指定与池关联的钩子地址。如果指定的钩子被黑客攻击或具有恶意，则提供的流动性可能被冻结或被窃取。

> **⚠️ 警告**：流动性提供者在使用此功能时需小心，**应仅使用经过审计并提供安全保障的钩子**。

---

## 核心源文件速查

| 文件 | 功能 |
|---|---|
| [`src/interfaces/IHooks.sol`](../../v4-core/src/interfaces/IHooks.sol) | 10 个钩子函数的接口定义 |
| [`src/libraries/Hooks.sol`](../../v4-core/src/libraries/Hooks.sol) | 权限标志常量、调度逻辑、callHook |
| [`src/libraries/LPFeeLibrary.sol`](../../v4-core/src/libraries/LPFeeLibrary.sol) | 动态费用标志、费用校验 |
| [`src/test/BaseTestHooks.sol`](../../v4-core/src/test/BaseTestHooks.sol) | 最小化基础钩子合约（继承用） |
| [`src/test/DynamicFeesTestHook.sol`](../../v4-core/src/test/DynamicFeesTestHook.sol) | 动态费用示例 |
| [`src/test/CustomCurveHook.sol`](../../v4-core/src/test/CustomCurveHook.sol) | 自定义 AMM 曲线示例 |
| [`src/test/FeeTakingHook.sol`](../../v4-core/src/test/FeeTakingHook.sol) | afterSwap 收费示例 |