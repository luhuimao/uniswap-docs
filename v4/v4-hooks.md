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

### `beforeSwap` 的内部调度逻辑

`Hooks.sol` 中 `beforeSwap` 的实现展示了调度的完整流程：

```solidity
function beforeSwap(
    IHooks self,
    PoolKey memory key,
    SwapParams memory params,
    bytes calldata hookData
) internal returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride) {
    amountToSwap = params.amountSpecified;

    // 自调用保护：钩子本身触发的交换不再触发钩子
    if (msg.sender == address(self)) return (amountToSwap, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self, abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // 返回值长度必须为 96 字节（bytes4 + BeforeSwapDelta + uint24）
        if (result.length != 96) InvalidHookResponse.selector.revertWith();

        // 动态费用池可通过 OVERRIDE_FEE_FLAG 覆写本次交换的费用
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        // 若钩子声明会返回 delta，则解析并调整 amountToSwap
        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                // 确保交换方向不因 delta 调整而改变（exactIn ↔ exactOut）
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0) {
                    HookDeltaExceedsSwapAmount.selector.revertWith();
                }
            }
        }
    }
}
```

### `afterSwap` 的内部调度逻辑

```solidity
function afterSwap(
    IHooks self,
    PoolKey memory key,
    SwapParams memory params,
    BalanceDelta swapDelta,
    bytes calldata hookData,
    BeforeSwapDelta beforeSwapHookReturn
) internal returns (BalanceDelta, BalanceDelta) {
    if (msg.sender == address(self)) return (swapDelta, BalanceDeltaLibrary.ZERO_DELTA);

    // 将 beforeSwap 中未处理的 unspecified delta 传递给 afterSwap 继续处理
    int128 hookDeltaSpecified   = beforeSwapHookReturn.getSpecifiedDelta();
    int128 hookDeltaUnspecified = beforeSwapHookReturn.getUnspecifiedDelta();

    if (self.hasPermission(AFTER_SWAP_FLAG)) {
        hookDeltaUnspecified += self.callHookWithReturnDelta(
            abi.encodeCall(IHooks.afterSwap, (msg.sender, key, params, swapDelta, hookData)),
            self.hasPermission(AFTER_SWAP_RETURNS_DELTA_FLAG)
        ).toInt128();
    }

    // 合并 hookDelta 并从 swapDelta 中扣除（由调用方承担钩子的 delta）
    BalanceDelta hookDelta;
    if (hookDeltaUnspecified != 0 || hookDeltaSpecified != 0) {
        hookDelta = (params.amountSpecified < 0 == params.zeroForOne)
            ? toBalanceDelta(hookDeltaSpecified, hookDeltaUnspecified)
            : toBalanceDelta(hookDeltaUnspecified, hookDeltaSpecified);
        swapDelta = swapDelta - hookDelta;
    }
    return (swapDelta, hookDelta);
}
```

### 调用工具函数

`Hooks.sol` 提供了两个底层调用辅助函数：

```solidity
// 用于不返回 delta 的钩子调用（beforeInitialize、beforeDonate 等）
function callHook(IHooks self, bytes memory data) internal returns (bytes memory result) {
    bool success;
    assembly ("memory-safe") {
        success := call(gas(), self, 0, add(data, 0x20), mload(data), 0, 0)
    }
    // 失败时携带原始 revert 信息抛出 HookCallFailed
    if (!success) CustomRevert.bubbleUpAndRevertWith(address(self), bytes4(data), HookCallFailed.selector);
    // ... 拷贝 returndata ...
    // 验证返回的函数选择器
    if (result.length < 32 || result.parseSelector() != data.parseSelector()) {
        InvalidHookResponse.selector.revertWith();
    }
}

// 用于返回 delta 的钩子调用（afterSwap、afterAddLiquidity 等）
function callHookWithReturnDelta(IHooks self, bytes memory data, bool parseReturn) internal returns (int256) {
    bytes memory result = callHook(self, data);
    if (!parseReturn) return 0;                         // 未声明 RETURNS_DELTA_FLAG，默认 0
    if (result.length != 64) InvalidHookResponse.selector.revertWith(); // bytes4 + int256
    return result.parseReturnDelta();
}
```

### 防止自调用的 `noSelfCall` 修饰符

```solidity
// 防止钩子合约自身触发的操作再次递归调用钩子
modifier noSelfCall(IHooks self) {
    if (msg.sender != address(self)) {
        _;
    }
}
```

这个修饰符用于不需要 delta 返回值的钩子（`beforeInitialize`、`afterInitialize`、`beforeDonate`、`afterDonate`），避免递归调用导致的无限循环。

---

## 动态费用机制：`LPFeeLibrary.sol`

> 源文件：[`src/libraries/LPFeeLibrary.sol`](../../v4-core/src/libraries/LPFeeLibrary.sol)

动态费用通过几个关键常量和函数实现：

```solidity
library LPFeeLibrary {

    /// @notice 费用 = 0x800000 时标识该池为动态费用池
    /// 注意：0x800000 > MAX_LP_FEE，因此它不是一个合法的静态费用值
    uint24 public constant DYNAMIC_FEE_FLAG  = 0x800000;

    /// @notice beforeSwap 返回的费用若要覆写当前费用，需设置此标志位（第23位）
    uint24 public constant OVERRIDE_FEE_FLAG = 0x400000;

    /// @notice 移除覆写标志时使用的掩码
    uint24 public constant REMOVE_OVERRIDE_MASK = 0xBFFFFF;

    /// @notice 最大 LP 费用：100%（以万分之一为单位，1_000_000 = 100%）
    uint24 public constant MAX_LP_FEE = 1_000_000;

    function isDynamicFee(uint24 self) internal pure returns (bool) {
        return self == DYNAMIC_FEE_FLAG;
    }

    function isOverride(uint24 self) internal pure returns (bool) {
        return self & OVERRIDE_FEE_FLAG != 0;
    }

    function removeOverrideFlagAndValidate(uint24 self) internal pure returns (uint24 fee) {
        fee = self & REMOVE_OVERRIDE_MASK; // 移除覆写标志
        if (fee > MAX_LP_FEE) revert LPFeeTooLarge(fee);
    }
}
```

**动态费用的两种使用方式**：

1. **永久性更新**：调用 `IPoolManager.updateDynamicLPFee(key, newFee)`，永久修改池的 LP 费用。
2. **单次覆写**：在 `beforeSwap` 返回携带 `OVERRIDE_FEE_FLAG` 的费用值（如 `0x400000 | newFee`），仅对本次交换生效。

---

## 实战代码示例

### 示例一：动态费用钩子

> 源文件：[`src/test/DynamicFeesTestHook.sol`](../../v4-core/src/test/DynamicFeesTestHook.sol)

```solidity
contract DynamicFeesTestHook is BaseTestHooks {
    uint24 internal fee;
    IPoolManager manager;

    /// @notice 池初始化后立即设置初始动态费用
    function afterInitialize(address, PoolKey calldata key, uint160, int24)
        external override returns (bytes4)
    {
        manager.updateDynamicLPFee(key, fee); // 永久性更新池费用
        return IHooks.afterInitialize.selector;
    }

    /// @notice 每次 swap 前更新费用（实现基于预言机/波动率的动态调整）
    function beforeSwap(address, PoolKey calldata key, SwapParams calldata, bytes calldata)
        external override returns (bytes4, BeforeSwapDelta, uint24)
    {
        manager.updateDynamicLPFee(key, fee); // 永久性更新
        return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
        // 若要仅对本次 swap 生效，可返回：
        // return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, fee | LPFeeLibrary.OVERRIDE_FEE_FLAG);
    }
}
```

### 示例二：自定义曲线钩子（替换集中流动性 AMM）

> 源文件：[`src/test/CustomCurveHook.sol`](../../v4-core/src/test/CustomCurveHook.sol)

该示例演示如何通过 `beforeSwapDelta` 完全接管交换逻辑，实现自定义 AMM 曲线（此处为 1:1 线性曲线）。

```solidity
contract CustomCurveHook is BaseTestHooks {
    IPoolManager immutable manager;

    function beforeSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        (Currency inputCurrency, Currency outputCurrency, uint256 amount) =
            _getInputOutputAndAmount(key, params);

        // 自定义曲线：1:1 线性兑换
        // 1. 从池中取出用户输入的代币
        manager.take(inputCurrency, address(this), amount);
        // 2. 向池中存入等量的输出代币
        outputCurrency.settle(manager, address(this), amount, false);

        // 3. 通过返回负的 amountSpecified，让 PoolManager "no-op" 原有集中流动性逻辑
        BeforeSwapDelta hookDelta = toBeforeSwapDelta(
            int128(-params.amountSpecified),  // specified delta：抵消原始金额
            int128(params.amountSpecified)    // unspecified delta：提供输出金额
        );
        return (IHooks.beforeSwap.selector, hookDelta, 0);
    }
}
```

> **核心原理**：通过将 `specified delta` 设置为 `-amountSpecified`，钩子将整个交换金额"吸收"，使 `PoolManager` 中的集中流动性路径成为 no-op，交换完全由钩子的自定义逻辑处理。

### 示例三：afterSwap 交易费用收取钩子

> 源文件：[`src/test/FeeTakingHook.sol`](../../v4-core/src/test/FeeTakingHook.sol)

```solidity
contract FeeTakingHook is BaseTestHooks {
    uint128 public constant SWAP_FEE_BIPS = 123;  // 1.23%
    uint128 public constant TOTAL_BIPS    = 10000;

    function afterSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, int128) {
        // 在 unspecified 代币（输出代币）上按比例计算并收取手续费
        bool specifiedTokenIs0 = (params.amountSpecified < 0 == params.zeroForOne);
        (Currency feeCurrency, int128 swapAmount) = specifiedTokenIs0
            ? (key.currency1, delta.amount1())
            : (key.currency0, delta.amount0());

        if (swapAmount < 0) swapAmount = -swapAmount;
        uint256 feeAmount = uint128(swapAmount) * SWAP_FEE_BIPS / TOTAL_BIPS;

        // 从 PoolManager 中取走手续费，发送到本合约（钩子金库）
        manager.take(feeCurrency, address(this), feeAmount);

        // 返回正数 delta：通知 PoolManager 钩子从用户输出中"取走"了 feeAmount
        return (IHooks.afterSwap.selector, feeAmount.toInt128());
    }
}
```

### 示例四：BaseTestHooks — 最小化基础合约

> 源文件：[`src/test/BaseTestHooks.sol`](../../v4-core/src/test/BaseTestHooks.sol)

在实际开发中，无需实现所有 10 个钩子函数。推荐继承 `BaseTestHooks`，只覆盖（`override`）你需要的函数：

```solidity
contract BaseTestHooks is IHooks {
    error HookNotImplemented();

    // 所有未覆写的函数默认 revert HookNotImplemented()
    function beforeSwap(
        address, PoolKey calldata, SwapParams calldata, bytes calldata
    ) external virtual returns (bytes4, BeforeSwapDelta, uint24) {
        revert HookNotImplemented();
    }

    function afterSwap(
        address, PoolKey calldata, SwapParams calldata, BalanceDelta, bytes calldata
    ) external virtual returns (bytes4, int128) {
        revert HookNotImplemented();
    }

    // ... 其他函数同理 ...
}
```

这样，`PoolManager` 只会调用在地址中标记为已实现的函数（通过权限位判断），未标记的函数不会被调用，因此 `HookNotImplemented` 永远不会触发。

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

在示例 `00C0`（`0000 0000 1100 0000`）中，仅有第 7 位（`beforeSwap`）和第 6 位（`afterSwap`）被设置为 `1`，表示该钩子仅实现这两个函数。

### 为什么将函数编码到地址中？

将钩子函数编码到合约地址本身的原因是**降低 Gas 成本**。

通过将函数实现直接编码到合约地址中，`PoolManager` 能够很快知道该执行哪些函数。这使得无效的函数调用可以完全跳过，从而降低了 Gas 消耗。

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

Uniswap 提供了以下工具来简化地址挖矿流程：

- **[HookMiner](https://github.com/Uniswap/v4-template)**：自动化地址挖矿过程，使开发者能够高效地找到其钩子的有效地址进行测试。
- **[V4 Hook Address Miner](https://uniswaphooks.com/)**（基于 Web 的 UI 工具）：允许开发者：
  - 选择他们希望在钩子中实现的功能
  - 自动生成一个有效地址所需的 `salt`
  - 部署他们的钩子合约，而无需手动挖找有效地址

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