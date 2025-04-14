---
title: Uniswap合约解析
aside: true
top_img: img/a6.png
cover: img/a6.png
date: 2023-11-25 20:44:32
updated: 2023-11-25 20:44:32
description: Uniswap是知名的DEFI，这里解读其核心合约
categories:
- 工作
- Uniswap
tags:
- Solidity
- 笔记
- 合约
---

# Uniswap合约

这里主要参考了文章：

[重新理解Uniswap V3：一次非常精彩的创新 - 碳链价值 (ccvalue.cn)](https://www.ccvalue.cn/article/1031408.html)

[UniswapV3简析(一) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/592696536)

[UniSwap V3协议浅析(下)-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1850737)

![img](https://pic1.zhimg.com/80/v2-4f8faa69a994cedefce7283d3b524c68_720w.webp)

Uniswap核心分为核心合约(core contracts )与外围合约(periphery contracts)两部分组成.

## 核心合约

核心合约主要包括 UniswapV3Pool, UniswapV3Factory,UniswapV3PoolDeployer等，其他合约大多为实现某个类型的某些功能。

### Factory合约

​	UniswapV3Factory的主要功能是提供创建pool的接口并且跟踪所有的pool。

合约最初声明当前工程合约的owner、费用的tickSpacing(刻度间距，如果为0则表示未启用，一经添加无法删除)、根据token与fee检索交易池的映射：

```solidity
    /// @inheritdoc IUniswapV3Factory
    address public override owner;

    /// @inheritdoc IUniswapV3Factory
    mapping(uint24 => int24) public override feeAmountTickSpacing;
    /// @inheritdoc IUniswapV3Factory
    mapping(address => mapping(address => mapping(uint24 => address))) public override getPool;
```

之后通过构造函数初始化的合约的owner以及三个TickSpace(没理解错的话一个tick和上一个tick价格差距0.01% 所以上下10个tick相当于0.1%左右的范围)：

```solidity
    constructor() {
        owner = msg.sender;
        emit OwnerChanged(address(0), msg.sender);

        feeAmountTickSpacing[500] = 10;
        emit FeeAmountEnabled(500, 10);
        feeAmountTickSpacing[3000] = 60;
        emit FeeAmountEnabled(3000, 60);
        feeAmountTickSpacing[10000] = 200;
        emit FeeAmountEnabled(10000, 200);
    }
```

上面定义的三个TickSpace与费率的关系如下所示：

![img](https://ask.qcloudimg.com/http-save/yehe-8423247/ac91653052761821e8ea58cc2aaab94e.png)

之后通过createPool来创建交易池，此时需要提供以下三个参数：

- fee:期望的费率
- tokenA：交易池中的两个Token之一
- tokenB：交易池中的两个Token之一

在createPool函数中首先会检查tokenA与tokenB是否是同一Token，之后将TokenA与TokenB根据地址进行升序排列为token0和token1(token0成为基础币而token1成为价值衡量币)，之后检查token0地址是否为空地址，之后根据费率检索TickSpace并检查TickSpace是否为0(构造函数会进行初始化一次)，之后检查当前新建的池子是否已经存在，之后通过deploy创建池子，然后新增池子记录，在新增记录时可以看到也提供了反向映射，这样做的好处是在减少后期检索时比较地址的成本，最后通过emit触发池子创建事件

```solidity
    /// @inheritdoc IUniswapV3Factory
    function createPool(
        address tokenA,
        address tokenB,
        uint24 fee
) external override noDelegateCall returns (address pool) {
        require(tokenA != tokenB);
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0));
        int24 tickSpacing = feeAmountTickSpacing[fee];
        require(tickSpacing != 0);
        require(getPool[token0][token1][fee] == address(0));
        pool = deploy(address(this), token0, token1, fee, tickSpacing);
        getPool[token0][token1][fee] = pool;
        // populate mapping in the reverse direction, deliberate choice to avoid the cost of comparing addresses
        getPool[token1][token0][fee] = pool;
        emit PoolCreated(token0, token1, fee, tickSpacing, pool);
    }
```

之后的setOwner函数用于更新工厂合约的owner，该函数只能由合约的owner调用，在更新时通过emit来触发owner变更事件：

```solidity
    /// @inheritdoc IUniswapV3Factory
    function setOwner(address _owner) external override {
        require(msg.sender == owner);
        emit OwnerChanged(owner, _owner);
        owner = _owner;
    }
```

enableFeeAmount函数用于新增tickSpacing(刻度间距)的费率，该函数只能由合约的owner调用，在函数中首先对参数进行合法性检查，在这里要求费率不得超过1000000(即:100%)，tickSpacing不得超过16384以防止tickSpacing过大，之后检查该费率是否已经存在，之后对给定tickSpacing设置费率，之后通过emit触发事件：

```solidity
    /// @inheritdoc IUniswapV3Factory
    function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {
        require(msg.sender == owner);
        require(fee < 1000000);
        // tick spacing is capped at 16384 to prevent the situation where tickSpacing is so large that
        // TickBitmap#nextInitializedTickWithinOneWord overflows int24 container from a valid tick
        // 16384 ticks represents a >5x price change with ticks of 1 bips
        require(tickSpacing > 0 && tickSpacing < 16384);
        require(feeAmountTickSpacing[fee] == 0);

        feeAmountTickSpacing[fee] = tickSpacing;
        emit FeeAmountEnabled(fee, tickSpacing);
    }
```

### PoolDeployer合约

​	UniswapV3PoolDeployer合约主要提供deploy函数来创建UniswapV3Pool智能合约并设置两个token信息，交易费用信息和tick的步长信息.

其只有一个deploy函数：

```solidity
    function deploy(
        address factory,
        address token0,
        address token1,
        uint24 fee,
        int24 tickSpacing
    ) internal returns (address pool) {
        parameters = Parameters({factory: factory, token0: token0, token1: token1, fee: fee, tickSpacing: tickSpacing});
        pool = address(new UniswapV3Pool{salt: keccak256(abi.encode(token0, token1, fee))}());
        delete parameters;
    }
```

deploy函数首先将参数再次作为参数初始化为一个Parameters的结构体变量，之后通过keccak256(abi.encode(token0, token1, fee)将token0、token1、fee作为参数得到一个哈希值，之后将其作为salt来创建合约，故而solidity会使用EVM的CREATE2指令来创建合约，这里使用CREATE2指令的一个好处就是，只要合约的bytecode及salt不变，那么创建出来的地址也将不变，也就是说每个交易池都有一个唯一的地址，之后清空结构体变量.

### Pool合约

![img](https://pic2.zhimg.com/80/v2-d336608dc0fdcad547358f2d26e41535_720w.webp)

​	Pool合约主要实现代币交易、流动性管理、交易手续费收取、Oracle管理等。Uniswap本质上是由N多个独立的流动池构成, 每个流动池由(token0,token1,fee)三个参数唯一进行标识, 通过这三个参数可以唯一检索出对应的Pool地址。

* token0：作为基准货币
* token1：作为价值衡量货币
* fee：手续费用

​	每一个流动池Pool实际上是一个流动性聚合器, 可以同时添加多个流动性仓位, 每个流动性仓位由(owner,tickLower,tickUpper)三个参数唯一标识，通过上面三个参数唯一检索出对应的流动性仓位Position：这里的仓位拥有者都是NonfungiblePositionManager合约, 而不是实际的用户地址。

* owner: 流动性仓位的拥有者地址(实际上就是NonfungiblePositionManager合约)
* tickLower: 流动性仓位做市区间最低价的tick索引
* tickUpper: 流动性仓位做市区间最高价的tick索引



首先，声明了合约中使用到的全局变量，之后定义了结构体Slot0(一个用于采样池子最新状况的结构，当前最新price、当前tick、协议费用、池子状态(开启或关闭))、结构体ProtocolFees等：

```solidity
    using LowGasSafeMath for uint256;
    using LowGasSafeMath for int256;
    using SafeCast for uint256;
    using SafeCast for int256;
    using Tick for mapping(int24 => Tick.Info);
    using TickBitmap for mapping(int16 => uint256);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;
    using Oracle for Oracle.Observation[65535];

    /// @inheritdoc IUniswapV3PoolImmutables
    address public immutable override factory;
    /// @inheritdoc IUniswapV3PoolImmutables
    address public immutable override token0;
    /// @inheritdoc IUniswapV3PoolImmutables
    address public immutable override token1;
    /// @inheritdoc IUniswapV3PoolImmutables
    uint24 public immutable override fee;

    /// @inheritdoc IUniswapV3PoolImmutables
    int24 public immutable override tickSpacing;      // 刻度间隔

    /// @inheritdoc IUniswapV3PoolImmutables
    uint128 public immutable override maxLiquidityPerTick; // 可使用范围内任何刻度的头寸流动性的最大金额

    struct Slot0 {
        // the current price
        uint160 sqrtPriceX96;
        // the current tick
        int24 tick;
        // the most-recently updated index of the observations array
        uint16 observationIndex;
        // the current maximum number of observations that are being stored
        uint16 observationCardinality;
        // the next maximum number of observations to store, triggered in observations.write
        uint16 observationCardinalityNext;
        // the current protocol fee as a percentage of the swap fee taken on withdrawal
        // represented as an integer denominator (1/x)%
        uint8 feeProtocol;
        // whether the pool is locked
        bool unlocked;
    }
    /// @inheritdoc IUniswapV3PoolState
    Slot0 public override slot0;

    /// @inheritdoc IUniswapV3PoolState
    uint256 public override feeGrowthGlobal0X128;
    /// @inheritdoc IUniswapV3PoolState
    uint256 public override feeGrowthGlobal1X128;

    // accumulated protocol fees in token0/token1 units
    struct ProtocolFees {
        uint128 token0;
        uint128 token1;
    }
    /// @inheritdoc IUniswapV3PoolState
    ProtocolFees public override protocolFees;

    /// @inheritdoc IUniswapV3PoolState
    uint128 public override liquidity;

    /// @inheritdoc IUniswapV3PoolState
    mapping(int24 => Tick.Info) public override ticks;
    /// @inheritdoc IUniswapV3PoolState
    mapping(int16 => uint256) public override tickBitmap;
    /// @inheritdoc IUniswapV3PoolState
    mapping(bytes32 => Position.Info) public override positions;
    /// @inheritdoc IUniswapV3PoolState
    Oracle.Observation[65535] public override observations;
```

修饰器lock用于提供锁机制来规避重入攻击，此方法还可以防止在初始化池之前调用函数，重入保护在整个合约中是必需的，因为我们经常使用余额检查来确定交互(如铸币、掉期和闪存)的支付状态：

```solidity
    /// @dev Mutually exclusive reentrancy protection into the pool to/from a method. This method also prevents entrance
    /// to a function before the pool is initialized. The reentrancy guard is required throughout the contract because
    /// we use balance checks to determine the payment status of interactions such as mint, swap and flash.
    modifier lock() {
        require(slot0.unlocked, 'LOK');
        slot0.unlocked = false;
        _;
        slot0.unlocked = true;
    }
```

修饰器onlyFactoryOwner用于检测函数的调用者是否为工程合约的owner地址

```solidity
    /// @dev Prevents calling a function from anyone except the address returned by IUniswapV3Factory#owner()
    modifier onlyFactoryOwner() {
        require(msg.sender == IUniswapV3Factory(factory).owner());
        _;
    }
```

构造函数用于初始化一个池子，其实完整的流程应该为：

UniswapV3Factory.sol(createPool)—>UniswapV3PoolDeployer.sol(deploy)—>UniswapV3Pool.sol(constructor)

```solidity
    constructor() {
        int24 _tickSpacing;
        (factory, token0, token1, fee, _tickSpacing) = IUniswapV3PoolDeployer(msg.sender).parameters();
        tickSpacing = _tickSpacing;

        maxLiquidityPerTick = Tick.tickSpacingToMaxLiquidityPerTick(_tickSpacing);
    }
```

checkTicks函数用于检测流动性的价格上限与流动性的价格下限的合法性：

```solidity
    /// @dev Common checks for valid tick inputs.
    function checkTicks(int24 tickLower, int24 tickUpper) private pure {
        require(tickLower < tickUpper, 'TLU');
        require(tickLower >= TickMath.MIN_TICK, 'TLM');
        require(tickUpper <= TickMath.MAX_TICK, 'TUM');
    }
```

函数_blockTimestamp用于返回截断后(32位)的区块时间戳信息：

```solidity
    /// @dev Returns the block timestamp truncated to 32 bits, i.e. mod 2**32. This method is overridden in tests.
    function _blockTimestamp() internal view virtual returns (uint32) {
        return uint32(block.timestamp); // truncation is desired
    }
```

balance0与balance1函数用于检索池子中token0与token1的资产数量：

```solidity
    /// @dev Get the pool's balance of token0
    /// @dev This function is gas optimized to avoid a redundant extcodesize check in addition to the returndatasize
    /// check
    function balance0() private view returns (uint256) {
        (bool success, bytes memory data) =
            token0.staticcall(abi.encodeWithSelector(IERC20Minimal.balanceOf.selector, address(this)));
        require(success && data.length >= 32);
        return abi.decode(data, (uint256));
    }

    /// @dev Get the pool's balance of token1
    /// @dev This function is gas optimized to avoid a redundant extcodesize check in addition to the returndatasize
    /// check
    function balance1() private view returns (uint256) {
        (bool success, bytes memory data) =
            token1.staticcall(abi.encodeWithSelector(IERC20Minimal.balanceOf.selector, address(this)));
        require(success && data.length >= 32);
        return abi.decode(data, (uint256));
    }
```

#### Initialize

initialize函数用于对交易池进行初始化，主要涉及池子价格、tick、协议费用等，这里的sqrtPriceX96为sqrt(amountToken1/amountToken0)Q64.96精度的定点数值，在这里首先检查池子价格是否未初始化，之后调用getTickAtSqrtRatio函数以sqrtPriceX96为参数来计算最大的tick，之后调用observations.initialize获取cardinality(基数)与cardinalityNext(下一个基数)的数值，之后对slot0进行初始化操作，完成交易池的创建，当然此时交易池里面还没有流动性：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    /// @dev not locked because it initializes unlocked
    function initialize(uint160 sqrtPriceX96) external override {
        require(slot0.sqrtPriceX96 == 0, 'AI');

        int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

        (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(_blockTimestamp());

        slot0 = Slot0({
            sqrtPriceX96: sqrtPriceX96,
            tick: tick,
            observationIndex: 0,
            observationCardinality: cardinality,
            observationCardinalityNext: cardinalityNext,
            feeProtocol: 0,
            unlocked: true
        });

        emit Initialize(sqrtPriceX96, tick);
    }
```

#### Mint

![img](https://pic1.zhimg.com/80/v2-afca012d34c2c5eb20a91c0fb75f7d20_720w.webp)

mint函数用于添加流动性（流动性实际是一直ERC20标准的代币，我猜测是为了随时跟踪各个池子的情况），相关参数如下：

- recipient - 流动性仓位的拥有者地址
- tickLower - 流动性仓位做市价格区间(也叫做流动性头寸）最低点位
- tickUpper - 流动性仓位做市价格区间最高点位
- amount - 要添加的流动性大小, 也就是L的值
- data - 回调函数参数

在这里首先检查增加的流动性的数量是否大于0，之后调用ModifyPositionParams初始化一个Position结构体的示例，之后再次调用_modifyPosition来调整仓位Position，该函数调用完之后会返回应投入的token0和token1的数量，之后获取当前池子之前的token0和token1的数量，之后将需要投入的token0和token1的数量传给NonfungiblePositionManager合约的uniswapV3MintCallback回调函数, 在这个函数里面完成真正的转账充值工作, 然后会检查是否成功收到转账, 如果出现问题系统会进行回滚，如果满足则通过emit触发事件：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    /// @dev noDelegateCall is applied indirectly via _modifyPosition
    function mint(
        address recipient,
        int24 tickLower,
        int24 tickUpper,
        uint128 amount,
        bytes calldata data
    ) external override lock returns (uint256 amount0, uint256 amount1) {
        require(amount > 0);
        (, int256 amount0Int, int256 amount1Int) =
            _modifyPosition(
                ModifyPositionParams({
                    owner: recipient,
                    tickLower: tickLower,
                    tickUpper: tickUpper,
                    liquidityDelta: int256(amount).toInt128()
                })
            );

        amount0 = uint256(amount0Int);
        amount1 = uint256(amount1Int);

        uint256 balance0Before;
        uint256 balance1Before;
        if (amount0 > 0) balance0Before = balance0();
        if (amount1 > 0) balance1Before = balance1();
        IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data);
        if (amount0 > 0) require(balance0Before.add(amount0) <= balance0(), 'M0');
        if (amount1 > 0) require(balance1Before.add(amount1) <= balance1(), 'M1');

        emit Mint(msg.sender, recipient, tickLower, tickUpper, amount, amount0, amount1);
    }
```

​	_modifyPosition会依据做市价格区间及其流动L的大小计算需要充值的token0,token1的数量, 假设当前流动池价格为Pc, 用户添加流动性仓位的价格区间的 [Pa, Pb] (Pa < Pb), 实际会分为三种情况：

1. 做市价格区间大于当前实际价格, 只能添加x token, 如下图:

![img](https://pic2.zhimg.com/80/v2-b207b879cc3f30fb692db160078f8349_720w.webp)

流动性L的计算公式为:

![img](https://pic1.zhimg.com/80/v2-37c1d5345a579fd4daa7c1b12d8f888c_720w.webp)

2. 做市价格区间小于当前实际价格, 只能添加y token, 如下图:

![img](https://pic2.zhimg.com/80/v2-f9e7a40ce6b0d5f4a6b0105218349c21_720w.webp)

流动性L的计算公式为:

![img](https://pic4.zhimg.com/80/v2-a13d4d9c1bbe5eddd382c9ec52e26a37_720w.webp)

3. 做市价格区间包含当前实际价格, 按一定比例分别添加x token和y token, 如下图:

![img](https://pic2.zhimg.com/80/v2-bad293f2a809ff76d9c371ca9eeec485_720w.webp)

流动性L的计算公式为:

![img](https://pic1.zhimg.com/80/v2-2e7a9da2eb7ab355cad505f8adf6f19c_720w.webp)

下面对上述涉及到的相关函数方法进行简易拆分：

ModifyPositionParams——用于存储Position(流动性仓位)相关的数据信息，包括流动性仓位所有者地址、流动性仓位的上下限、流动性的改动：

```solidity
    struct ModifyPositionParams {
        // the address that owns the position
        address owner;
        // the lower and upper tick of the position
        int24 tickLower;
        int24 tickUpper;
        // any change in liquidity
        int128 liquidityDelta;
    }
```

_modifyPosition——用于更新当前Position，在这里首先通过checkTicks来检查流动性上下限是否满足条件，之后将其读入内存(Slot0 memory _slot0 = slot0;)，后续通过SLOAD直接访问而不用重新去加载LOAD，从而节省gas，紧接着又调用了_updatePosition来更新position，紧接着检查当前流动性的变更是否为0，如果不为0则检查当前的tick是否小于ticklower，如果是则所有的token1将转变为token0，之后计算token0的变更；如果当前tick在两者之间，则通过observations.write增加Oracle条目，之后计算amout0和amount1的增量，最后当前tick范围内的流动性总量；如果tick超过了tickupper则此时所有的token0将转变为token1，之后计算amount1的变更，需要注意的是这里的amount0与amount1为int256类型，也就是说这里的amount0与amount1这两个返回值可正可负，如果为正则表示流动性提供者需要给池子给予的数量，为负数则表示池子需要给流动性提供者给予的数量：

```solidity
    /// @dev Effect some changes to a position
    /// @param params the position details and the change to the position's liquidity to effect
    /// @return position a storage pointer referencing the position with the given owner and tick range
    /// @return amount0 the amount of token0 owed to the pool, negative if the pool should pay the recipient
    /// @return amount1 the amount of token1 owed to the pool, negative if the pool should pay the recipient
    function _modifyPosition(ModifyPositionParams memory params)
        private
        noDelegateCall
        returns (
            Position.Info storage position,
            int256 amount0,
            int256 amount1
)
    {
        checkTicks(params.tickLower, params.tickUpper);

        Slot0 memory _slot0 = slot0; // SLOAD for gas optimization

        position = _updatePosition(
            params.owner,
            params.tickLower,
            params.tickUpper,
            params.liquidityDelta,
            _slot0.tick
        );

        if (params.liquidityDelta != 0) {
            if (_slot0.tick < params.tickLower) {
                // current tick is below the passed range; liquidity can only become in range by crossing from left to
                // right, when we'll need _more_ token0 (it's becoming more valuable) so user must provide it
                amount0 = SqrtPriceMath.getAmount0Delta(
                    TickMath.getSqrtRatioAtTick(params.tickLower),
                    TickMath.getSqrtRatioAtTick(params.tickUpper),
                    params.liquidityDelta
                );
            } else if (_slot0.tick < params.tickUpper) {
                // current tick is inside the passed range
                uint128 liquidityBefore = liquidity; // SLOAD for gas optimization

                // write an oracle entry
                (slot0.observationIndex, slot0.observationCardinality) = observations.write(
                    _slot0.observationIndex,
                    _blockTimestamp(),
                    _slot0.tick,
                    liquidityBefore,
                    _slot0.observationCardinality,
                    _slot0.observationCardinalityNext
                );

                amount0 = SqrtPriceMath.getAmount0Delta(
                    _slot0.sqrtPriceX96,
                    TickMath.getSqrtRatioAtTick(params.tickUpper),
                    params.liquidityDelta
                );
                amount1 = SqrtPriceMath.getAmount1Delta(
                    TickMath.getSqrtRatioAtTick(params.tickLower),
                    _slot0.sqrtPriceX96,
                    params.liquidityDelta
                );

                liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);
            } else {
                // current tick is above the passed range; liquidity can only become in range by crossing from right to
                // left, when we'll need _more_ token1 (it's becoming more valuable) so user must provide it
                amount1 = SqrtPriceMath.getAmount1Delta(
                    TickMath.getSqrtRatioAtTick(params.tickLower),
                    TickMath.getSqrtRatioAtTick(params.tickUpper),
                    params.liquidityDelta
                );
            }
        }
    }
```

_updatePosition函数，该函数代码如下，在这里首先通过get函数来获取用户的position信息，之后检查流动性变更是否为0，如果不为零则获取当前时间戳信息并通过observations.observeSingle来获取请求时间点的Oracle数据，之后更新position对应的ticklower/tickupper的数据，之后检查flippedLower是否是第一次被引用，或者移除了所有引用(移除流动性操作时)，如果是则更新tick位图，之后更新position数据，并清除之前的记录：

```solidity
    /// @dev Gets and updates a position with the given liquidity delta
    /// @param owner the owner of the position
    /// @param tickLower the lower tick of the position's tick range
    /// @param tickUpper the upper tick of the position's tick range
    /// @param tick the current tick, passed to avoid sloads
    function _updatePosition(
        address owner,
        int24 tickLower,
        int24 tickUpper,
        int128 liquidityDelta,
        int24 tick
    ) private returns (Position.Info storage position) {
        position = positions.get(owner, tickLower, tickUpper);

        uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128; // SLOAD for gas optimization
        uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128; // SLOAD for gas optimization

        // if we need to update the ticks, do it
        bool flippedLower;
        bool flippedUpper;
        if (liquidityDelta != 0) {
            uint32 time = _blockTimestamp();
            (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
                observations.observeSingle(
                    time,
                    0,
                    slot0.tick,
                    slot0.observationIndex,
                    liquidity,
                    slot0.observationCardinality
                );

            flippedLower = ticks.update(
                tickLower,
                tick,
                liquidityDelta,
                _feeGrowthGlobal0X128,
                _feeGrowthGlobal1X128,
                secondsPerLiquidityCumulativeX128,
                tickCumulative,
                time,
                false,
                maxLiquidityPerTick
            );
            flippedUpper = ticks.update(
                tickUpper,
                tick,
                liquidityDelta,
                _feeGrowthGlobal0X128,
                _feeGrowthGlobal1X128,
                secondsPerLiquidityCumulativeX128,
                tickCumulative,
                time,
                true,
                maxLiquidityPerTick
            );

            if (flippedLower) {
                tickBitmap.flipTick(tickLower, tickSpacing);
            }
            if (flippedUpper) {
                tickBitmap.flipTick(tickUpper, tickSpacing);
            }
        }

        (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
            ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);

        position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

        // clear any tick data that is no longer needed
        if (liquidityDelta < 0) {
            if (flippedLower) {
                ticks.clear(tickLower);
            }
            if (flippedUpper) {
                ticks.clear(tickUpper);
            }
        }
    }
```

uniswapV3MintCallback——即回调函数，该函数会将指定数量的token0与token1发送到合约中去：

```solidity
    function uniswapV3MintCallback(
        uint256 amount0Owed,
        uint256 amount1Owed,
        bytes calldata data
    ) external override {
        address sender = abi.decode(data, (address));

        emit MintCallback(amount0Owed, amount1Owed);
        if (amount0Owed > 0)
            IERC20Minimal(IUniswapV3Pool(msg.sender).token0()).transferFrom(sender, msg.sender, amount0Owed);
        if (amount1Owed > 0)
            IERC20Minimal(IUniswapV3Pool(msg.sender).token1()).transferFrom(sender, msg.sender, amount1Owed);
    }
```

#### Burn

burn函数用于移除流动性，和之前的mint函数正好相逆，相关参数如下：

- tickLower - 做市价格区间最低点位, 用于检索对应流动性仓位
- tickUpper - 做市价格区间最高点位, 用于检索对应流动性仓位
- amount - 要减少的流动性数量, 也就是L值

函数首先初始化一个ModifyPositionParams结构体类型的示例，之后调用_modifyPosition来修改position，因为减少了流动性, 所以必然会空闲出相应数量的token, 程序会把空闲出来的token数量保存到对应仓位的position.tokensOwed0,position.tokensOwed1成员变量中,需要强调的是, 这里仅仅只是记录空闲出来的token数量, 而并不会把这些token返还给客户：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    /// @dev noDelegateCall is applied indirectly via _modifyPosition
    function burn(
        int24 tickLower,
        int24 tickUpper,
        uint128 amount
    ) external override lock returns (uint256 amount0, uint256 amount1) {
        (Position.Info storage position, int256 amount0Int, int256 amount1Int) =
            _modifyPosition(
                ModifyPositionParams({
                    owner: msg.sender,
                    tickLower: tickLower,
                    tickUpper: tickUpper,
                    liquidityDelta: -int256(amount).toInt128()
                })
            );

        amount0 = uint256(-amount0Int);
        amount1 = uint256(-amount1Int);

        if (amount0 > 0 || amount1 > 0) {
            (position.tokensOwed0, position.tokensOwed1) = (
                position.tokensOwed0 + uint128(amount0),
                position.tokensOwed1 + uint128(amount1)
            );
        }

        emit Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1);
    }
```

#### Collect

collect函数用于提取闲置token及其手续费，相关参数如下所示：

- recipient - 接收返还的用户地址
- tickLower - 做市价格区间最低点位, 用于检索对应流动性仓位
- tickUpper - 做市价格区间最高点位, 用于检索对应流动性仓位
- amount0Requested - 想要提取的token0数量
- amount1Requested - 想要提取的token1数量

当用户进行减少流动性的操作后, 空闲出来的token数量会被保存到对应仓位的position.tokensOwed0,position.tokensOwed1成员变量中, 另外就是用户充当LP赚取的手续费也会保存在这俩成员变量中, 本函数会真正将相应数量的token返还给用户。首先通过positions.get获取当前用户position信息，之后根据参数调整需要收取的手续费，并将手续费发送给流动性提供者指定的用于接受手续费的地址：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    function collect(
        address recipient,
        int24 tickLower,
        int24 tickUpper,
        uint128 amount0Requested,
        uint128 amount1Requested
) external override lock returns (uint128 amount0, uint128 amount1) {
        // we don't need to checkTicks here, because invalid positions will never have non-zero tokensOwed{0,1}
        Position.Info storage position = positions.get(msg.sender, tickLower, tickUpper);

        amount0 = amount0Requested > position.tokensOwed0 ? position.tokensOwed0 : amount0Requested;
        amount1 = amount1Requested > position.tokensOwed1 ? position.tokensOwed1 : amount1Requested;

        if (amount0 > 0) {
            position.tokensOwed0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
            position.tokensOwed1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1);
    }
```

#### Swap

swap函数主要用于处理交易，提供token交易兑换功能, 算是较为复杂的部分, 因为交易过程中可能会跨越多个流动性仓位, 算法有一定的复杂性，相关参数说明：

- recipient - 接收兑换输出的地址
- zeroForOne - 兑换方向, 当true时兑换方向为:token0->token1, 当false时兑换方向为:token1到token0
- amountSpecified - 兑换数量, 当数值为正数代表是输入token的数量, 当数值为负数代表想达到的输出token数量
- sqrtPriceLimitX96 - 价格限制, 根据方向最终价格不能大于或者小于该值
- data - 回调函数参数

基本逻辑就是:

1.已知输入token的数量, 计算出可以输出多少数量的token, 及其兑换结束后价格移动到什么点位

2.已知输出token的数量, 计算出需要多少数量的输入token, 及其兑换结束后价格移动到什么点位

分为四种情况处理:

1. (zeroForOne:true, amountSpecified:正数) ==>> 我用amountSpecified数量的token0, 最多可以兑换多少数量的token1?

2. (zeroForOne:true, amountSpecified:负数) ==>> 我需要兑换abs(amountSpecified)数量的token1, 最少需要准备多少数量的token0?

3. (zeroForOne:false, amountSpecified:正数) ==>> 我用amountSpecified数量的token1, 最多可以兑换多少数量的token0?

4. (zeroForOne:false, amountSpecified:负数) ==>> 我需要兑换abs(amountSpecified)数量的token0, 最少需要准备多少数量的token1?

真正在token交易过程中, 可能会涉及路径选择问题, 比如ETH->DAI, 可以直接在ETH/DAI流动池兑换, 也可以利用ETH->USDC->DAI这个路径, 通过ETH/USDC, USDC/DAI两个流动池接力完成兑换

至于采取哪个路径, 由链下前端决定, 链下前端会根据情况计算出最优路径, 关于程序中路径的编码如下图所示:

![img](https://pic3.zhimg.com/80/v2-2312c9a0d89d39f02a448a225dc6e562_720w.webp)

当最终计算好要兑换的输入输出token数量以后, swap函数会首先将用户需求的token发送给用户, 然后会回调位于SwapRouter合约中的回调函数uniswapV3SwapCallback, 在uniswapV3SwapCallback函数中最终会用ERC20合约的transferFrom方法将用户的token转帐到流动池合约中, 至此完成兑换过程.



在这里首先检查交易数量是否为0，之后将交易前的数据保存在内存中，之后检查是否有正在进行的token交易操作，如果没有则根据交易方向检查价格是否满足在条件，之后更新slot0.unlocked的状态，之后缓存交易的数据，之后检查交换的数量是否大于0，并将其赋值于exactInput，之后存储交易状态信息：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
) external override noDelegateCall returns (int256 amount0, int256 amount1) {
        require(amountSpecified != 0, 'AS');

        Slot0 memory slot0Start = slot0;

        require(slot0Start.unlocked, 'LOK');
        require(
            zeroForOne
                ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
                : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
            'SPL'
        );

        slot0.unlocked = false;

        SwapCache memory cache =
            SwapCache({
                liquidityStart: liquidity,
                blockTimestamp: _blockTimestamp(),
                feeProtocol: zeroForOne ? (slot0Start.feeProtocol % 16) : (slot0Start.feeProtocol >> 4),
                secondsPerLiquidityCumulativeX128: 0,
                tickCumulative: 0,
                computedLatestObservation: false
            });

        bool exactInput = amountSpecified > 0;

        SwapState memory state =
            SwapState({
                amountSpecifiedRemaining: amountSpecified,
                amountCalculated: 0,
                sqrtPriceX96: slot0Start.sqrtPriceX96,
                tick: slot0Start.tick,
                feeGrowthGlobalX128: zeroForOne ? feeGrowthGlobal0X128 : feeGrowthGlobal1X128,
                protocolFee: 0,
                liquidity: cache.liquidityStart
            });
```

只有还有剩余的input/output同时没有超过价格限制则正常进行交易操作，在这里示例化了一个StepComputations变量，该变量存储了交易过程中的信息：

```solidity
    struct StepComputations {
        // the price at the beginning of the step
        uint160 sqrtPriceStartX96;
        // the next tick to swap to from the current tick in the swap direction
        int24 tickNext;
        // whether tickNext is initialized or not
        bool initialized;
        // sqrt(price) for the next tick (1/0)
        uint160 sqrtPriceNextX96;
        // how much is being swapped in in this step
        uint256 amountIn;
        // how much is being swapped out
        uint256 amountOut;
        // how much fee is being paid in
        uint256 feeAmount;
    }
```

之后获取交易的起始价格，然后通过位图找到下一个可选的交易价格，这里的下一个可选的交易价格可能会到下一个流动性中，也可能还是在本流动性中，之后需要确保我们不会超过最小和最大tick，因为tick位图不知道这些边界：

```solidity
        // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
        while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
            StepComputations memory step;

            step.sqrtPriceStartX96 = state.sqrtPriceX96;

            (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
                state.tick,
                tickSpacing,
                zeroForOne
            );

            // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
            if (step.tickNext < TickMath.MIN_TICK) {
                step.tickNext = TickMath.MIN_TICK;
            } else if (step.tickNext > TickMath.MAX_TICK) {
                step.tickNext = TickMath.MAX_TICK;
            }
```

之后获取下一个tick的价格，之后计算要交换目标价格、输入/输出量、费用等：

```solidity
            // get the price for the next tick
            step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);
        
            // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
            (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
                state.sqrtPriceX96,
                (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
                    ? sqrtPriceLimitX96
                    : step.sqrtPriceNextX96,
                state.liquidity,
                state.amountSpecifiedRemaining,
                fee
            );
```

如果exactInput为true则表示input不为负数，之后更新amountSpecifiedRemaining(剩余期望兑换的数量)与amountCalculated(已经兑换的数量)，如果exactInput为false则表示input为负数，之后更新amountSpecifiedRemaining(剩余期望兑换的数量)与amountCalculated(已经兑换的数量)，不过需要注意的是这里的符号有所变化，运算逻辑也有所改变，之后检查协议费用是否开启，如果开启则计算欠费多少，并减少feetAmount，增加protocolFee：

```solidity
            if (exactInput) {
                state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
                state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
            } else {
                state.amountSpecifiedRemaining += step.amountOut.toInt256();
                state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
            }

            // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
            if (cache.feeProtocol > 0) {
                uint256 delta = step.feeAmount / cache.feeProtocol;
                step.feeAmount -= delta;
                state.protocolFee += uint128(delta);
            }

            // update global fee tracker
            if (state.liquidity > 0)
                state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
```

当更新到下一个价格时更新tick，检查tick是否已经初始化，如果是则进行tick过渡处理，具体的方法是检查占位符的值，并使用第一次穿过初始化刻度时的实际值来替换它，之后更新tick的值，如果tokenIn被耗尽，则直接计算当前价格对应的tick：

```solidity
            // shift tick if we reached the next price
            if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
                // if the tick is initialized, run the tick transition
                if (step.initialized) {
                    // check for the placeholder value, which we replace with the actual value the first time the swap
                    // crosses an initialized tick
                    if (!cache.computedLatestObservation) {
                        (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                            cache.blockTimestamp,
                            0,
                            slot0Start.tick,
                            slot0Start.observationIndex,
                            cache.liquidityStart,
                            slot0Start.observationCardinality
                        );
                        cache.computedLatestObservation = true;
                    }
                    int128 liquidityNet =
                        ticks.cross(
                            step.tickNext,
                            (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                            (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                            cache.secondsPerLiquidityCumulativeX128,
                            cache.tickCumulative,
                            cache.blockTimestamp
                        );
                    // if we're moving leftward, we interpret liquidityNet as the opposite sign
                    // safe because liquidityNet cannot be type(int128).min
                    if (zeroForOne) liquidityNet = -liquidityNet;

                    state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
                }

                state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
            } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
                // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
                state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
            }
```

 之后当tick改变时写入一个oracle条目，否则只更新价格，如果流动性发生变化则更新流动性，以及更新费用等，最后进行转账和费用收取操作：

```solidity
        // update tick and write an oracle entry if the tick change
        if (state.tick != slot0Start.tick) {
            (uint16 observationIndex, uint16 observationCardinality) =
                observations.write(
                    slot0Start.observationIndex,
                    cache.blockTimestamp,
                    slot0Start.tick,
                    cache.liquidityStart,
                    slot0Start.observationCardinality,
                    slot0Start.observationCardinalityNext
                );
            (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
                state.sqrtPriceX96,
                state.tick,
                observationIndex,
                observationCardinality
            );
        } else {
            // otherwise just update the price
            slot0.sqrtPriceX96 = state.sqrtPriceX96;
        }

        // update liquidity if it changed
        if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;

        // update fee growth global and, if necessary, protocol fees
        // overflow is acceptable, protocol has to withdraw before it hits type(uint128).max fees
        if (zeroForOne) {
            feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
            if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
        } else {
            feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
            if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
        }

        (amount0, amount1) = zeroForOne == exactInput
            ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
            : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);

        // do the transfers and collect payment
        if (zeroForOne) {
            if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));

            uint256 balance0Before = balance0();
            IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
            require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
        } else {
            if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

            uint256 balance1Before = balance1();
            IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
            require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
        }

        emit Swap(msg.sender, recipient, amount0, amount1, state.sqrtPriceX96, state.liquidity, state.tick);
        slot0.unlocked = true;
    }
```

#### Flash

flash函数主要用于实现闪电贷，相关参数说明如下：

- recipient - 想借token的用户地址,比如可以是用户自己开发的合约的地址
- amount0 - 用户想借入的token0数量
- amount1 - 用户想借入的token1数量
- data - 回调函数参数

在该函数中首先计算借贷需要扣除的手续费，之后记录当前的余额，之后将借贷的token发送给借贷方，之后调用借贷方地址的回调函数将用户传入的data参数传递给该回调函数，之后再次记录调用完后的余额，之后检查回调函数调用完后余额的数量，每一个token的余额只可多不可少，之后计算借出的token0与token1的数量以及费用等：

```solidity
    /// @inheritdoc IUniswapV3PoolActions
    function flash(
        address recipient,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external override lock noDelegateCall {
        uint128 _liquidity = liquidity;
        require(_liquidity > 0, 'L');

        uint256 fee0 = FullMath.mulDivRoundingUp(amount0, fee, 1e6); //用户需要支付的借贷手续费
        uint256 fee1 = FullMath.mulDivRoundingUp(amount1, fee, 1e6); //用户需要支付的借贷手续费
        uint256 balance0Before = balance0(); //借贷前的金额
        uint256 balance1Before = balance1(); //借贷前的金额
        if (amount0 > 0) TransferHelper.safeTransfer(token0, recipient, amount0); //借出token给用户
        if (amount1 > 0) TransferHelper.safeTransfer(token1, recipient, amount1); //借出token给用户
        IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(fee0, fee1, data); //用户应该自己开发合约并实现此回调函数,注意参数是需要支付的手续费数量
        uint256 balance0After = balance0(); //用户已经连本带利归还后的余额
        uint256 balance1After = balance1(); //用户已经连本带利归还后的余额
        require(balance0Before.add(fee0) <= balance0After, 'F0'); //检查用户是不是已经连本带利足额还款, 没还就回滚整个交易
        require(balance1Before.add(fee1) <= balance1After, 'F1'); //检查用户是不是已经连本带利足额还款, 没还就回滚整个交易


        // sub is safe because we know balanceAfter is gt balanceBefore by at least fee
        uint256 paid0 = balance0After - balance0Before;
        uint256 paid1 = balance1After - balance1Before;

        if (paid0 > 0) {
            uint8 feeProtocol0 = slot0.feeProtocol % 16;
            uint256 fees0 = feeProtocol0 == 0 ? 0 : paid0 / feeProtocol0;
            if (uint128(fees0) > 0) protocolFees.token0 += uint128(fees0);
            feeGrowthGlobal0X128 += FullMath.mulDiv(paid0 - fees0, FixedPoint128.Q128, _liquidity);
        }
        if (paid1 > 0) {
            uint8 feeProtocol1 = slot0.feeProtocol >> 4;
            uint256 fees1 = feeProtocol1 == 0 ? 0 : paid1 / feeProtocol1;
            if (uint128(fees1) > 0) protocolFees.token1 += uint128(fees1);
            feeGrowthGlobal1X128 += FullMath.mulDiv(paid1 - fees1, FixedPoint128.Q128, _liquidity);
        }

        emit Flash(msg.sender, recipient, amount0, amount1, paid0, paid1);
    }
```

#### ProtocalFee

一般没人用似乎。

setFeeProtocol用于设置协议的费用份额的分母，相关参数说明如下：

- feeProtocol0：池中token0的新协议费用
- feeProtocol1：池中token1的新协议费用

```solidity
    /// @inheritdoc IUniswapV3PoolOwnerActions
    function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
        require(
            (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
                (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
        );
        uint8 feeProtocolOld = slot0.feeProtocol;
        slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
        emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
    }
```

collectProtocol函数用于收取协议费用，相关参数说明如下：

- recipient：接受协议费用的地址
- amount0Requested：在token0中收取的协议费用
- amount1Requested：在token1中收取的协议费用

```solidity
    /// @inheritdoc IUniswapV3PoolOwnerActions
    function collectProtocol(
        address recipient,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        if (amount0 > 0) {
            if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
            if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit CollectProtocol(msg.sender, recipient, amount0, amount1);
    }
```

#### Snapshot

snapshotCumulativesInside函数用于返回分时累计的快照、每个流动性的秒数和分时范围内的秒数，快照只能与其他快照进行比较，这些快照在头寸存在的时间段内拍摄，如果在拍摄第一个快照和拍摄第二个快照之间的整个时间段内没有持有仓位，则无法比较快照，相关参数如下：

- tickLower：范围下限
- tickUpper：范围上限 

返回值说明：

- tickCumulativeInside：对应范围的刻度累加器的快照
- secondsPerLiquidityInsideX128：范围内每个流动性的秒数快照 
- secondsInside：范围内每个流动性的秒数快照

```solidity
    /// @inheritdoc IUniswapV3PoolDerivedState
    function snapshotCumulativesInside(int24 tickLower, int24 tickUpper)
        external
        view
        override
        noDelegateCall
        returns (
            int56 tickCumulativeInside,
            uint160 secondsPerLiquidityInsideX128,
            uint32 secondsInside
        )
    {
        checkTicks(tickLower, tickUpper);

        int56 tickCumulativeLower;
        int56 tickCumulativeUpper;
        uint160 secondsPerLiquidityOutsideLowerX128;
        uint160 secondsPerLiquidityOutsideUpperX128;
        uint32 secondsOutsideLower;
        uint32 secondsOutsideUpper;

        {
            Tick.Info storage lower = ticks[tickLower];
            Tick.Info storage upper = ticks[tickUpper];
            bool initializedLower;
            (tickCumulativeLower, secondsPerLiquidityOutsideLowerX128, secondsOutsideLower, initializedLower) = (
                lower.tickCumulativeOutside,
                lower.secondsPerLiquidityOutsideX128,
                lower.secondsOutside,
                lower.initialized
            );
            require(initializedLower);

            bool initializedUpper;
            (tickCumulativeUpper, secondsPerLiquidityOutsideUpperX128, secondsOutsideUpper, initializedUpper) = (
                upper.tickCumulativeOutside,
                upper.secondsPerLiquidityOutsideX128,
                upper.secondsOutside,
                upper.initialized
            );
            require(initializedUpper);
        }

        Slot0 memory _slot0 = slot0;

        if (_slot0.tick < tickLower) {
            return (
                tickCumulativeLower - tickCumulativeUpper,
                secondsPerLiquidityOutsideLowerX128 - secondsPerLiquidityOutsideUpperX128,
                secondsOutsideLower - secondsOutsideUpper
            );
        } else if (_slot0.tick < tickUpper) {
            uint32 time = _blockTimestamp();
            (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
                observations.observeSingle(
                    time,
                    0,
                    _slot0.tick,
                    _slot0.observationIndex,
                    liquidity,
                    _slot0.observationCardinality
                );
            return (
                tickCumulative - tickCumulativeLower - tickCumulativeUpper,
                secondsPerLiquidityCumulativeX128 -
                    secondsPerLiquidityOutsideLowerX128 -
                    secondsPerLiquidityOutsideUpperX128,
                time - secondsOutsideLower - secondsOutsideUpper
            );
        } else {
            return (
                tickCumulativeUpper - tickCumulativeLower,
                secondsPerLiquidityOutsideUpperX128 - secondsPerLiquidityOutsideLowerX128,
                secondsOutsideUpper - secondsOutsideLower
            );
        }
    }
```

observe函数用于请求前N秒之前的历史数据，这里的参数secondsAgos为一个动态数组，故而可以一次请求多个历史数据，返回变量tickCumulatives和 liquidityCumulatives也是动态数组，用于记录请求参数中对应时间戳的tick index累积值和流动性累积值，之后调用observations.observe()处理数据：

```solidity
    /// @inheritdoc IUniswapV3PoolDerivedState
    function observe(uint32[] calldata secondsAgos)
        external
        view
        override
        noDelegateCall
        returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s)
{
        return
            observations.observe(
                _blockTimestamp(),
                secondsAgos,
                slot0.tick,
                slot0.observationIndex,
                liquidity,
                slot0.observationCardinality
            );
    }
```

observe代码如下所示，该函数通过遍历请求参数获取每一个请求时间点的Oracle数据，数据获取通过observeSingle来实现：

```solidity
    /// @notice Returns the accumulator values as of each time seconds ago from the given time in the array of `secondsAgos`
    /// @dev Reverts if `secondsAgos` > oldest observation
    /// @param self The stored oracle array
    /// @param time The current block.timestamp
    /// @param secondsAgos Each amount of time to look back, in seconds, at which point to return an observation
    /// @param tick The current tick
    /// @param index The index of the observation that was most recently written to the observations array
    /// @param liquidity The current in-range pool liquidity
    /// @param cardinality The number of populated elements in the oracle array
    /// @return tickCumulatives The tick * time elapsed since the pool was first initialized, as of each `secondsAgo`
    /// @return secondsPerLiquidityCumulativeX128s The cumulative seconds / max(1, liquidity) since the pool was first initialized, as of each `secondsAgo`
    function observe(
        Observation[65535] storage self,
        uint32 time,
        uint32[] memory secondsAgos,
        int24 tick,
        uint16 index,
        uint128 liquidity,
        uint16 cardinality
    ) internal view returns (int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) {
        require(cardinality > 0, 'I');

        tickCumulatives = new int56[](secondsAgos.length);
        secondsPerLiquidityCumulativeX128s = new uint160[](secondsAgos.length);
        for (uint256 i = 0; i < secondsAgos.length; i++) {
            (tickCumulatives[i], secondsPerLiquidityCumulativeX128s[i]) = observeSingle(
                self,
                time,
                secondsAgos[i],
                tick,
                index,
                liquidity,
                cardinality
            );
        }
    }
```

## 外围合约

### NonfungiblePositionManger合约

​	NonfungiblePositionManager合约是直接与链下前端进行交互的外围合约, 主要对外提供比如创建流动池, 添加流动性, 删除流动性等API接口, 同时也会为每个流动性仓位生成一个唯一的NFT给到用户, 可以说是Uniswap协议中最为重要的一个管理器，这里需要再次强调一下, 就是NonfungiblePositionManager合约可以同时管理多个流动池Pool(每个流动池包含一个交易对), 其由(token0,token1,fee)唯一标识, 而每个流动池又是一个流动性聚合器其同时包含多个流动性仓位, 而每个流动性仓位又可以由(owner,tickLower,tickUpper)唯一进行标识.

NonfungiblePositionManager合约对外提供一些重要的API:

1. **createAndInitializePoolIfNecessary**, 部署一个新的流动池合约
2. **selfPermitAllowed/selfPermit**, 授权合约有转账相应token的权限
3. **mint**, 给指定流动池添加一个新的流动性仓位, 并且生成一个代表此仓位的NFT给到用户
4. **increaseLiquidity**, 给指定流动池某个已经存在的流动性仓位增加流动性
5. **decreaseLiquidity**, 给指定流动池某个已经存在的流动性仓位减少流动性
6. **collect**, 提取闲置token及其手续费
7. **multicall**, 将多个合约调用打包进一次合约调用中, 这个其实是为了达到类似数据库事务(transaction)的效果, 一个事务应该满足:
   1. 原子性(Atomicity): 事务中的全部操作在数据库中是不可分割的, 要么全部完成, 要么全部不执行
   2. 一致性(Consistency): 几个并行执行的事务, 其执行结果必须与按某一顺序串行执行的结果相一致
   3. 隔离性(Isolation): 事务的执行不受其他事务的干扰, 事务执行的中间结果对其他事务必须是透明的
   4. 持久性(Durability): 对于任意已提交事务, 系统必须保证该事务对数据库的改变不被丢失, 即使数据库出现故障

合约开头首先定义了相关数据结构，包括：position结构体、用于交易池ID检索的mapping、Pool Keys检索的mapping等：

```solidity
    // details about the uniswap position
    struct Position {
        // the nonce for permits
        uint96 nonce;
        // the address that is approved for spending this token
        address operator;
        // the ID of the pool with which this token is connected
        uint80 poolId;
        // the tick range of the position
        int24 tickLower;
        int24 tickUpper;
        // the liquidity of the position
        uint128 liquidity;
        // the fee growth of the aggregate position as of the last action on the individual position
        uint256 feeGrowthInside0LastX128;
        uint256 feeGrowthInside1LastX128;
        // how many uncollected tokens are owed to the position, as of the last computation
        uint128 tokensOwed0;
        uint128 tokensOwed1;
    }

    /// @dev IDs of pools assigned by this contract
    mapping(address => uint80) private _poolIds;

    /// @dev Pool keys by pool ID, to save on SSTOREs for position data
    mapping(uint80 => PoolAddress.PoolKey) private _poolIdToPoolKey;

    /// @dev The token ID position data
    mapping(uint256 => Position) private _positions;

    /// @dev The ID of the next token that will be minted. Skips 0
    uint176 private _nextId = 1;
    /// @dev The ID of the next pool that is used for the first time. Skips 0
    uint80 private _nextPoolId = 1;

    /// @dev The address of the token descriptor contract, which handles generating token URIs for position tokens
    address private immutable _tokenDescriptor;
```

构造函数进行初始化操作：

```solidity
    constructor(
        address _factory,
        address _WETH9,
        address _tokenDescriptor_
) ERC721Permit('Uniswap V3 Positions NFT-V1', 'UNI-V3-POS', '1') PeripheryImmutableState(_factory, _WETH9) {
        _tokenDescriptor = _tokenDescriptor_;
    }
```

positions用于检索与tokenid相关position信息：

```solidity
    /// @inheritdoc INonfungiblePositionManager
    function positions(uint256 tokenId)
        external
        view
        override
        returns (
            uint96 nonce,
            address operator,
            address token0,
            address token1,
            uint24 fee,
            int24 tickLower,
            int24 tickUpper,
            uint128 liquidity,
            uint256 feeGrowthInside0LastX128,
            uint256 feeGrowthInside1LastX128,
            uint128 tokensOwed0,
            uint128 tokensOwed1
        )
    {
        Position memory position = _positions[tokenId];
        require(position.poolId != 0, 'Invalid token ID');
        PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
        return (
            position.nonce,
            position.operator,
            poolKey.token0,
            poolKey.token1,
            poolKey.fee,
            position.tickLower,
            position.tickUpper,
            position.liquidity,
            position.feeGrowthInside0LastX128,
            position.feeGrowthInside1LastX128,
            position.tokensOwed0,
            position.tokensOwed1
        );
    }
```

#### Mint

mint函数给指定流动池添加一个新的流动性仓位, 并且生成一个代表此仓位的NFT给到用户

```solidity
function mint(MintParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity( //此函数内部真正添加流动性,并完成token的转账充值
        AddLiquidityParams({
            token0: params.token0, //唯一标识一个流动池
            token1: params.token1, //唯一标识一个流动池
            fee: params.fee, //唯一标识一个流动池
            recipient: address(this), //唯一标识该池中的一个流动性仓位
            tickLower: params.tickLower, //唯一标识该池中的一个流动性仓位
            tickUpper: params.tickUpper, //唯一标识该池中的一个流动性仓位
            amount0Desired: params.amount0Desired, //要充值的token0数量
            amount1Desired: params.amount1Desired, //要充值的token1数量
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );
    _mint(params.recipient, (tokenId = _nextId++)); //生成一个代表流动性仓位的NFT给到用户
    bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);
    // idempotent set
    uint80 poolId =
        cachePoolKey(
            address(pool),
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee})
        );
    _positions[tokenId] = Position({ //将NFT与对应的流动性仓位进行关联
        nonce: 0,
        operator: address(0),
        poolId: poolId, //该NFT所代表的流动性仓位属于哪个流动池
        tickLower: params.tickLower, //该NFT所代表的流动性仓位做市区间最低点位
        tickUpper: params.tickUpper, //该NFT所代表的流动性仓位做市区间最高点位
        liquidity: liquidity, //该NFT所代表的流动性仓位的流动性大小, 也就是L值
        feeGrowthInside0LastX128: feeGrowthInside0LastX128, //计算手续费相关
        feeGrowthInside1LastX128: feeGrowthInside1LastX128, //计算手续费相关
        tokensOwed0: 0,
        tokensOwed1: 0
    });
    emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
}
```

tokenURI函数用于为position管理者生成描述特定TokenID的URI(源自ERC721):

```solidity
    function tokenURI(uint256 tokenId) public view override(ERC721, IERC721Metadata) returns (string memory) {
        require(_exists(tokenId));
        return INonfungibleTokenPositionDescriptor(_tokenDescriptor).tokenURI(this, tokenId);
    }
```

#### Burn

burn函数用于销毁TokenID：

```solidity
    /// @inheritdoc INonfungiblePositionManager
    function burn(uint256 tokenId) external payable override isAuthorizedForToken(tokenId) {
        Position storage position = _positions[tokenId];
        require(position.liquidity == 0 && position.tokensOwed0 == 0 && position.tokensOwed1 == 0, 'Not cleared');
        delete _positions[tokenId];
        _burn(tokenId);
    }
```

#### increaseLiquidity

increaseLiquidity函数用于给指定流动池某个已经存在的流动性仓位增加流动性

```solidity
function increaseLiquidity(IncreaseLiquidityParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    Position storage position = _positions[params.tokenId]; //根据传入的 NFT id 获取对应的流动性仓位信息
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId]; //获取该流动性仓位所属的流动池
    IUniswapV3Pool pool;
    (liquidity, amount0, amount1, pool) = addLiquidity( //向该流动池添加流动性, 不在赘述
        AddLiquidityParams({
            token0: poolKey.token0,
            token1: poolKey.token1,
            fee: poolKey.fee,
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min,
            recipient: address(this)
        })
    );
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);
    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey); //获取该流动池中对应的这个流动性仓位
    position.tokensOwed0 += uint128( //更新这个流动性仓位之前获取的做市手续费
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );
    position.tokensOwed1 += uint128( //更新这个流动性仓位之前获取的做市手续费
        FullMath.mulDiv(
            feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );
    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128; //更新手续费计算相关
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128; //更新手续费计算相关
    position.liquidity += liquidity; //更新L值
    emit IncreaseLiquidity(params.tokenId, liquidity, amount0, amount1);
}
```

#### decreaseLiquidity

decreaseLiquidity函数用于给指定流动池某个已经存在的流动性仓位减少流动性

```solidity
function decreaseLiquidity(DecreaseLiquidityParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    checkDeadline(params.deadline)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.liquidity > 0);
    Position storage position = _positions[params.tokenId]; //根据传入的 NFT id 获取对应的流动性仓位信息
    uint128 positionLiquidity = position.liquidity;
    require(positionLiquidity >= params.liquidity);
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId]; //获取该流动性仓位所属的流动池key
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey)); //获取该流动性仓位所属的流动池合约地址
    (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity); //调用该流动池合约的burn函数,真正完成减少流动性
    require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper); //获取该流动性仓位的key
    // this is now updated to the current transaction
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey); //从指定流动池中获取该流动性仓位的手续费相关信息
    position.tokensOwed0 +=
        uint128(amount0) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        ); //减少流动性后,会空闲出多余token,记录之,同时也会更新之前的做市手续费收入
    position.tokensOwed1 +=
        uint128(amount1) +
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            )
        ); //减少流动性后,会空闲出多余token,记录之,同时也会更新之前的做市手续费收入
    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128; //更新手续费计算相关
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128; //更新手续费计算相关
    // subtraction is safe because we checked positionLiquidity is gte params.liquidity
    position.liquidity = positionLiquidity - params.liquidity; //更新L值
    emit DecreaseLiquidity(params.tokenId, params.liquidity, amount0, amount1);
}
```

#### collect

collect函数用于提取闲置token及其手续费

```solidity
function collect(CollectParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)
    returns (uint256 amount0, uint256 amount1)
{
    require(params.amount0Max > 0 || params.amount1Max > 0);
    // allow collecting to the nft position manager address with address 0
    address recipient = params.recipient == address(0) ? address(this) : params.recipient;
    Position storage position = _positions[params.tokenId]; //根据传入的 NFT id 获取对应的流动性仓位信息
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId]; //根据流动性仓位信息找所属的流动池
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey)); //获取该流动池合约地址
    (uint128 tokensOwed0, uint128 tokensOwed1) = (position.tokensOwed0, position.tokensOwed1); //真正可提取的token数量
    // trigger an update of the position fees owed and fee growth snapshots if it has any liquidity
    if (position.liquidity > 0) {
        pool.burn(position.tickLower, position.tickUpper, 0); //这里并不是减少流动性,而仅仅只是为了更新手续费
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) =
            pool.positions(PositionKey.compute(address(this), position.tickLower, position.tickUpper));
        tokensOwed0 += uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        ); //把最新的可提取的手续费收入也累加进来
        tokensOwed1 += uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        ); //把最新的可提取的手续费收入也累加进来
        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    }
    // compute the arguments to give to the pool#collect method
    (uint128 amount0Collect, uint128 amount1Collect) =
        (
            params.amount0Max > tokensOwed0 ? tokensOwed0 : params.amount0Max,
            params.amount1Max > tokensOwed1 ? tokensOwed1 : params.amount1Max
        );
    // the actual amounts collected are returned
    (amount0, amount1) = pool.collect( //调用流动池合约的collect函数,真正完成转账提取token的操作
        recipient, //用户收钱地址
        position.tickLower, //用于标识对应的流动性仓位
        position.tickUpper, //用于标识对应的流动性仓位
        amount0Collect, //期望提取的token数量
        amount1Collect  //期望提取的token数量
    );
    // sometimes there will be a few less wei than expected due to rounding down in core, but we just subtract the full amount expected
    // instead of the actual amount so we can burn the token
    (position.tokensOwed0, position.tokensOwed1) = (tokensOwed0 - amount0Collect, tokensOwed1 - amount1Collect);
    emit Collect(params.tokenId, recipient, amount0Collect, amount1Collect);
}
```

#### muticall

multicall, 将多个合约调用打包进一次合约调用中, 这是事务性的关键操作

```solidity
function multicall(bytes[] calldata data) external payable override returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        (bool success, bytes memory result) = address(this).delegatecall(data[i]);
        if (!success) {
            // Next 5 lines from https://ethereum.stackexchange.com/a/83577
            if (result.length < 68) revert();
            assembly {
                result := add(result, 0x04)
            }
            revert(abi.decode(result, (string)));
        }
        results[i] = result;
    }
}
```

可以看到逻辑其实很简单, 就是在一个for循环中依次调用之前打包的各个合约调用, 这里有一个非常关键的技术要点就是delegatecall, delegatecall的作用是当用户A通过合约B来delegatecall合约C的时候, 执行的是合约C的函数, 但是语境仍是合约B的: msg.sender是A的地址, 并且如果函数改变一些状态变量, 产生的效果会作用于合约B的变量上, 如下图:

![img](https://pic1.zhimg.com/80/v2-3846ce5ac5e1c16279a891c373a696c0_720w.webp)

### SWAPROUTER合约

SwapRouter合约是直接与链下前端进行交互的外围合约, 主要对外交易兑换功能, 此合约可以看成用户的交易代理合约, 可以协助用户完成一些相对复杂的兑换操作, 用户<=\=>SwapRouter合约<=\=>UniswapV3Pool合约SwapRouter合约主要封装了四个与交易相关的API接口对外提供服务, 分别是:

1. **exactInputSingle**: 两个token之间进行兑换, 已知输入token的数量, 输出token的数量函数内部会自动进行计算

2. **exactInput**: 两个token之间进行兑换, 已知输入token的数量, 支持复杂路径, 比如ETH->DAI, 可以直接在ETH/DAI流动池兑换, 也可以利用ETH->USDC->DAI这个路径, 通过ETH/USDC, USDC/DAI两个流动池接力完成兑换, 至于采取哪个路径, 由链下前端决定, 链下前端会根据情况计算出最优路径.

3. **exactOutputSingle**: 两个token之间进行兑换, 已知输出token的数量, 输入token的数量函数内部会自动进行计算

4. **exactOutput**: 两个token之间进行兑换, 已知输出token的数量, 支持复杂路径(实际并没有实现)

需要说明的是：

1. 当创建流动池的时候就已经确保了池内的token0地址小于token1地址, 所以当用户想用tokenIn兑换tokenOut的时候, 也就是tokenIn=>tokenOut, 那么需要判断它俩地址的大小, 当tokenIn<tokenOut的时候, tokenIn等于池内的token0, tokenOut等于池内的token1, 也就是token0=>token1. 当tokenIn>tokenOut的时候, tokenIn等于池内的token1, tokenOut等于池内的token0, 也就是token1=>token0
2. 对于每一个流动池, 池内的token0是基础货币, token1是计价货币, 池内的价格指的是token0的价格, 以token1计价, 所以用token0兑换token1(token0=>token1), 可以理解为卖出操作, 对于卖出而言, sqrtPriceLimitX96指的是可以接受的最低卖出价格. 而用token1兑换token0(token1=>token0), 可以理解为买入操作, 对于买入而言, sqrtPriceLimitX96指的是可以接受的最高买入价格. 最后需要再次强调的是一个交易对在流动池里面只会以交易对中两个token合约地址大小来确定顺序, 而不会考虑日常的习惯, 比如交易对LINK/USDT, 假如LINK合约地址大于USDT合约地址, 那么它们在流动池里面实际是USDT/LINK, 也就是说token0是USDT, token1是LINK.

#### getpool

getPool函数用于根据tokenA、TokenB以及fee来获取对应的池子信息：

```solidity
    /// @dev Returns the pool for the given token pair and fee. The pool contract may or may not exist.
    function getPool(
        address tokenA,
        address tokenB,
        uint24 fee
) private view returns (IUniswapV3Pool) {
        return IUniswapV3Pool(PoolAddress.computeAddress(factory, PoolAddress.getPoolKey(tokenA, tokenB, fee)));
    }
```

#### SwapCallBack

uniswapV3SwapCallback函数用于回调操作：

```solidity
/// @inheritdoc IUniswapV3SwapCallback
    function uniswapV3SwapCallback(
        int256 amount0Delta,
        int256 amount1Delta,
        bytes calldata _data
    ) external override {
        require(amount0Delta > 0 || amount1Delta > 0); // swaps entirely within 0-liquidity regions are not supported
        SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));
        (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
        CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);
        (bool isExactInput, uint256 amountToPay) =
            amount0Delta > 0
                ? (tokenIn < tokenOut, uint256(amount0Delta))
                : (tokenOut < tokenIn, uint256(amount1Delta));
        if (isExactInput) {
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        } else {
            // either initiate the next swap or pay
            if (data.path.hasMultiplePools()) {
                data.path = data.path.skipToken();
                exactOutputInternal(amountToPay, msg.sender, 0, data);
            } else {
                amountInCached = amountToPay;
                tokenIn = tokenOut; // swap in/out because exact output swaps are reversed
                pay(tokenIn, data.payer, msg.sender, amountToPay);
            }
        }
    }
```

#### exactInputSingle/exactInput

exactInputSingle/exactInput函数用于计算将一种指定数量的token兑换成另一种token时可以做多获得多少的输出量：

```solidity
/// @inheritdoc ISwapRouter
    function exactInputSingle(ExactInputSingleParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 amountOut)
    {
        amountOut = exactInputInternal(
            params.amountIn,
            params.recipient,
            params.sqrtPriceLimitX96,
            SwapCallbackData({path: abi.encodePacked(params.tokenIn, params.fee, params.tokenOut), payer: msg.sender})
        );
        require(amountOut >= params.amountOutMinimum, 'Too little received');
    }
    /// @inheritdoc ISwapRouter
    function exactInput(ExactInputParams memory params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 amountOut)
    {
        address payer = msg.sender; // 第一个交易对肯定需要用户支付token
        while (true) {
            bool hasMultiplePools = params.path.hasMultiplePools(); //路径中是否包含多个交易对
            // the outputs of prior swaps become the inputs to subsequent ones
            params.amountIn = exactInputInternal(
                params.amountIn,
                hasMultiplePools ? address(this) : params.recipient, // 如果是复杂路径交易,中间token由合约自己先代收
                0,
                SwapCallbackData({
                    path: params.path.getFirstPool(), // only the first pool in the path is necessary
                    payer: payer
                })
            );
            // decide whether to continue or terminate
            if (hasMultiplePools) {
                payer = address(this); // 如果是复杂路径交易,中间token由合约自己先代收,所以也需要合约自己支付到下一个流动池
                params.path = params.path.skipToken();
            } else {
                amountOut = params.amountIn;
                break;
            }
        }
        require(amountOut >= params.amountOutMinimum, 'Too little received');
    }
```

这里调用的函数exactInputInternal代码如下，它是exactInputSingle的具体实现：

```solidity
    /// @dev Performs a single exact input swap
    function exactInputInternal(
        uint256 amountIn,
        address recipient,
        uint160 sqrtPriceLimitX96,
        SwapCallbackData memory data
    ) private returns (uint256 amountOut) {
        // allow swapping to the router address with address 0
        if (recipient == address(0)) recipient = address(this);
        (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool(); //路径解码
        bool zeroForOne = tokenIn < tokenOut; //根据地址大小决定池内是token0->token1还是token1->token0
        (int256 amount0, int256 amount1) =
            getPool(tokenIn, tokenOut, fee).swap(
                recipient, //收款地址
                zeroForOne,
                amountIn.toInt256(), //输入token的数量
                sqrtPriceLimitX96 == 0
                    ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                    : sqrtPriceLimitX96, //价格限制,卖设置最低价,买设置最高价
                abi.encode(data)
            );
        return uint256(-(zeroForOne ? amount1 : amount0)); //输出token数量用负数表示
    }
```

#### exactOutputSingle/exactOutput

exactOutputSingle/exactOutput用于根据指定的最小输出量来计算需要投入token的量：

```solidity
    /// @inheritdoc ISwapRouter
    function exactOutputSingle(ExactOutputSingleParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 amountIn)
    {
        // avoid an SLOAD by using the swap return data
        amountIn = exactOutputInternal(
            params.amountOut,
            params.recipient,
            params.sqrtPriceLimitX96,
            SwapCallbackData({path: abi.encodePacked(params.tokenOut, params.fee, params.tokenIn), payer: msg.sender})
        );
        require(amountIn <= params.amountInMaximum, 'Too much requested');
        // has to be reset even though we don't use it in the single hop case
        amountInCached = DEFAULT_AMOUNT_IN_CACHED;
    }
    
       function exactOutput(ExactOutputParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 amountIn)
    {
        // it's okay that the payer is fixed to msg.sender here, as they're only paying for the "final" exact output
        // swap, which happens first, and subsequent swaps are paid for within nested callback frames
        exactOutputInternal(
            params.amountOut,
            params.recipient,
            0,
            SwapCallbackData({path: params.path, payer: msg.sender})
        );
        amountIn = amountInCached;
        require(amountIn <= params.amountInMaximum, 'Too much requested');
        amountInCached = DEFAULT_AMOUNT_IN_CACHED;
    }
```

exactOutputInternal代码如下所示：

```solidity
     function exactOutputInternal(
        uint256 amountOut,
        address recipient,
        uint160 sqrtPriceLimitX96,
        SwapCallbackData memory data
    ) private returns (uint256 amountIn) {
        // allow swapping to the router address with address 0
        if (recipient == address(0)) recipient = address(this);
        (address tokenOut, address tokenIn, uint24 fee) = data.path.decodeFirstPool(); //路径解码
        bool zeroForOne = tokenIn < tokenOut; //根据地址大小决定池内是token0->token1还是token1->token0
        (int256 amount0Delta, int256 amount1Delta) =
            getPool(tokenIn, tokenOut, fee).swap(
                recipient,
                zeroForOne,
                -amountOut.toInt256(), //负数表示的是输出token数量
                sqrtPriceLimitX96 == 0
                    ? (zeroForOne ? TickMath.MIN_SQRT_RATIO + 1 : TickMath.MAX_SQRT_RATIO - 1)
                    : sqrtPriceLimitX96,
                abi.encode(data)
            );
        uint256 amountOutReceived;
        (amountIn, amountOutReceived) = zeroForOne
            ? (uint256(amount0Delta), uint256(-amount1Delta))
            : (uint256(amount1Delta), uint256(-amount0Delta));
        // it's technically possible to not receive the full output amount,
        // so if no price limit has been specified, require this possibility away
        if (sqrtPriceLimitX96 == 0) require(amountOutReceived == amountOut);
    }
```

### 其他合约

**TickLens.sol**, 此工具合约主要用于查询给定的流动池的所有流动性仓位的流动性大小, 这些信息将用于填充Uniswap链下前端信息网站上展示的流动性深度图

**Quoter.sol**, 此工具合约主要用于模拟真实交易, 获取实际交易输入输出的token数量, 用户在真实交易前, 链下前端需要预先计算出用户输入token能够预期兑换的输出token数, 但是这个计算工作只有链上的流动池合约自己能做到, 而且流动池合约中的swap函数都是会更改合约状态的external函数, 需要消耗gas费, 那么就需要把这个操作当作view/pure函数来使用, 本合约为了实现这个目的而存在. 备注: 新版本此合约已经升级成最新的QuoterV2版本, 新版本合约可以查询更多信息, 比如gas评估等