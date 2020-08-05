# YFII 流动性挖矿合约安全性研究

> https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.md
> 
> **Available in [ [English](https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.en.md) | [中文](https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.md) ]**

[YFII](https://yfii.finance/) 是一个新型去中心化 DeFi 矿池，应社区小伙伴邀请，[安比实验室](https://secbit.io)于 2020 年 7 月 27 日至 8 月 2 日对 YFII 智能合约进行了安全性研究。

分析对象为下列合约：

- YFII Pool 1: 0xb81D3cB2708530ea990a287142b82D058725C092
- YFII Pool 2: 0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a
- YFII Token: 0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83
- BPT Token: 0x16cAC1403377978644e78769Daa49d8f6B6CF565

初步分析结果表明，以上四个合约并未包含致命安全漏洞。安比实验室希望通过本文对研究过程做一个记录和总结，**Token 价格、经济模型、其他外部合约模块、以及未来新的合约不在本文讨论内**。

## YFII 和 YFI 是什么

Yearn Finance 是一个 DeFi 收益聚合器，于 7 月 17 日推出治理 Token YFI 后因其新颖的分发机制和治理方案迅速引爆了流动性挖矿市场。而 YFII 是对 YFI 项目的分叉，遵循 YIP-8 （Yearn Finance 8 号改进提案），实施了类似比特币的减半机制。

## YFII 相对 YFI 做了哪些改动

目前，YFII 合约代码均直接 Fork 自 Yearn Finance，做了较小的改动以支持 YFII Token 的定期减半分发。

下表为 YFII 涉及的合约与 YFI 合约对应关系及地址。

|      合约      |                        YFII 对应地址                         |                YFI 对应地址                |
| :------------: | :----------------------------------------------------------: | :----------------------------------------: |
|     Pool1 (YearnRewards)      | [0xb81D3cB2708530ea990a287142b82D058725C092](https://etherscan.io/address/0xb81D3cB2708530ea990a287142b82D058725C092#code) | [0x0001FB050Fe7312791bF6475b96569D83F695C9f](https://etherscan.io/address/0x0001FB050Fe7312791bF6475b96569D83F695C9f#code) |
|     Pool2 (YearnRewards)     |          [0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a](https://etherscan.io/address/0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a#code)          | [0x033E52f513F9B98e129381c6708F9faA2DEE5db5](https://etherscan.io/address/0x033E52f513F9B98e129381c6708F9faA2DEE5db5#code) |
| YFI/YFII Token (ERC20) |          [0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83](https://etherscan.io/address/0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83#code)          | [0x0bc529c00C6401aEF6D220BE8C6Ea1667F6Ad93e](https://etherscan.io/address/0x0bc529c00C6401aEF6D220BE8C6Ea1667F6Ad93e#code) |
|   BPT Token (Balancer)   |          [0x16cAC1403377978644e78769Daa49d8f6B6CF565](https://etherscan.io/address/0x16cAC1403377978644e78769Daa49d8f6B6CF565#code)          | [0x95c4b6c7cff608c0ca048df8b81a484aa377172b](https://etherscan.io/address/0x95c4b6c7cff608c0ca048df8b81a484aa377172b#code) |

YFI/YFII Token 为项目治理 Token 合约，二者实现一致，具体为一个带 mint 和简单 governance 功能的标准 ERC-20 Token。

BPT Token 为 Balancer Pool Token 合约，是做市商的流动性证明 Token，实际由自动做市协议 Balancer 的 [BFactory](https://etherscan.io/address/0x9424b1412450d0f8fc2255faf6046b98213b76bd#code) 入口合约创建，因此二者实现完全一致。该合约代码此前由 Trail of Bits 和 Consensys Diligence [进行过审计](https://docs.balancer.finance/protocol/security/audits)。

Pool1 和 Pool2 是用于分发治理 Token 的流动性挖矿合约，Pool1 和 Pool2 的代码实现一致，均被称为 YearnRewards 合约，而 YFII 相对于 YFI 的改动即在这该合约中。

YFII Pool1 和 Pool2 合约相对于原始代码新增了 `checkStart()` 和 `checkhalve()` 两个修饰符函数，分别用于**控制挖矿开始时间**，和**治理 Token 周期性地产量减半**。

## YFII & YFI 核心合约简析

YFII 和 YFI 流动性挖矿的核心合约代码 YearnRewards 实际源自于 Synthetix 项目的 [Unipool](https://github.com/Synthetixio/Unipool/blob/master/contracts/Unipool.sol)，原本用于奖励在 Uniswap 上为 ETH/sETH 交易对提供流动性的做市商 SNX Token，该代码之前经过 Sigma Prime [审计](https://github.com/sigp/public-audits/blob/master/synthetix/unipool/review.pdf)。

基于 YearnRewards 的流动性挖矿整个流程可以分为以下几步：

1. 具有 `RewardDistribution` 权限的地址，预先通过调用 YearnRewards 合约的`notifyRewardAmount()` 函数，设置奖励数额，而对应金额的 YFI Token 应由 YFI minter 转入 YearnRewards 合约中。
2. 矿工向 YearnRewards 合约指定的目标 DeFi 合约（可为自动做市商DEX、或借贷协议）提供流动性（通常为存入稳定币），拿到对应的流动性证明 Token（通常也称为 Pool Token），该 Token 可以用于换回资产以及赚取利息或手续费收益。
3. 矿工将得到的 Pool Token 通过调用 `stake()` 函数存入 YearnRewards 合约中，合约自动根据 Stake 时长和矿工存入资金规模占资金池总规模的大小来计算矿工应得的奖励。
4. 矿工可随时提走自己的应有奖励（YFI Token）以及之前存入的 Pool Token。

通常一个 YearnRewards 合约专门用于单个特定 DeFi 项目的流动性挖矿，如 Pool1 对接 Curve 项目的 y 池，Pool2 对接 Balancer 上的 YFI-DAI Pool。

## 一些发现

前面提到 YFII 相对于 YFI 的改动，代码改动整体较小。

新增两个修饰器函数用于约束 `stake()` `withdraw()` `getReward()` 三个主要功能函数。

```js
    modifier checkhalve(){
        if (block.timestamp >= periodFinish) {
            initreward = initreward.mul(50).div(100); 
            yfi.mint(address(this),initreward);

            rewardRate = initreward.div(DURATION);
            periodFinish = block.timestamp.add(DURATION);
            emit RewardAdded(initreward);
        }
        _;
    }

    modifier checkStart(){
        require(block.timestamp > starttime,"not start");
        _;
    }
```

`notifyRewardAmount()` 函数中新增了一行代码，用于在 notify 的同时直接控制 YFI Token 合约 mint （增发）指定数量的 Token 到当前 YearnRewards 合约，将其作为奖励用于分发。因此，Pool1 和 Pool2 合约必须是 YFII Token 合约的 minter。

这让 YFII 与 YFI 在 Token 分发细节逻辑上稍有不同。YFI 每期奖励的分发都需要由特定地址负责设置金额并转入 Token。而 YFII 除了第一期开始前执行了 `notifyRewardAmount()` 操作，之后会随着用户的调用，产量自动定期减半。

```js
    function notifyRewardAmount(uint256 reward)
        external
        onlyRewardDistribution
        updateReward(address(0))
    {
        if (block.timestamp >= periodFinish) {
            rewardRate = reward.div(DURATION);
        } else {
            uint256 remaining = periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardRate);
            rewardRate = reward.add(leftover).div(DURATION);
        }
        yfi.mint(address(this),reward); // added in YFII
        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp.add(DURATION);
        emit RewardAdded(reward);
    }
```

另外，在与社区开发者 Madao 和 gaojin 讨论代码细节的过程中，Madao 提到 Token 产量自动减半的执行需要依赖 `checkhalve()` 函数的执行，实际则依赖用户与合约交互，执行时间无法精准控制到上一个周期的结尾，减半发生时间会与预期时间存在一定的时间差，并且合约减半实际发生时间很大概率晚于预期时间。

特别地，合约计算奖励时会将两个周期间多出来的时间差计算在内，导致给每个用户计算的奖励值会略高于预期值，产生了一定误差。进而我们发现，只要误差存在，理论上最后一个从 Pool 中提取 reward 的人可能无法正常提现。这是因为合约在减半的同时 Mint YFII Token 至 Pool 合约。由于前面误差的存在，**导致合约中用户账面收益高于实际 Mint 出来的 Token 数量**。误差的大小计算方法为每个周期结束时间与下次减半发生实际发生时间之间的时间差 Delta 乘以减半后的 rewardRate。

![](https://i.imgur.com/JGULj1o.png)

根据上图测算，平均延时 60 秒减半，累计误差在 1 个 YFII 以内。只要能控制时间误差足够小，再加上持续有下一周期的 Token 作为补充，因此该误差问题影响较小。

另外，[@ThinkingETH](https://twitter.com/ThinkingETH/status/1290633339158687746) 提醒 `withdraw()` 函数不应该加上 `checkhalve()` 修饰器。如果 Pool 合约地址被故意或意外移除了 YFII token 的 minter 权限，那么用户将无法正常调用 `withdraw()` 函数因为 `mint()` 执行有可能会失败。

由于 YFII 已经将 token 的 owner 设为 0 地址（详见下一节），Pool1 和 Pool2 应该永远是 YFII token 的 minter，所以这个风险对于 YFII token 目前不存在。但作为智能合约代码来讲，这里的确应该实现得更严谨和更健壮，否则许多用户的资金可能处在高度风险中。尤其是很多进一步 fork YFII 代码的项目，如果开发者不了解这里面的细节，则很可能酿成悲剧。[@DoveyWan](https://twitter.com/DoveyWan/status/1290541716290670592) 和 [@oli_vdb](https://twitter.com/oli_vdb/status/1290370855709573122)  也提到了类似的安全事件。

## YFII 管理员权限处理

YFI 类 Token 都存在铸币（Mint）接口，具有 mint 权限的地址可以增发 Token。YFI 类 Token 还存在 Governance 管理员，具有权限添加和删除 Minter。通常理想情况下这些地址特殊权限地址应该为多签合约或其他专门合约。

另外 YearnRewards 合约有 `rewardDistribution` 权限地址，用于调用 `notifyRewardAmount()` 函数设置奖励金额。YearnRewards 合约还存在 `owner` 权限地址，用于设置 `rewardDistribution` 地址。

目前，YFII 项目的做法是将 YFII Token 的 Governance 管理员、Pool1 和 Pool2 的 `rewardDistribution` 均设为了 0 地址。管理员权限销毁记录可参见 https://burn.yfii.finance/ 。经查验，管理员权限销毁属实。目前只有 Pool1 和 Pool2 两个合约地址具有 YFII Token 的 mint 权限，属于为了实现周期性减半的**必要权限**，且未来**无法被滥用**。

特别值得一提地是，原版 YFI Token 代码实现中，并未给 `addMinter()` 该特权函数添加 Event，导致普通用户无法方便地查看究竟合约有多少 minter。当心，**这让各种类 YFI 项目非常容易藏入后门**。

经查验，YFII Token 合约总共[只有两条 `addMinter()` 记录](https://api.blockchair.com/ethereum/transactions?q=input_hex(^983b2d56),recipient(0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83))，分别为 Pool1 和 Pool2 合约添加 mint 权限，**未引入多余的 minter**。

## 总结

YFI 整体是一次非常有意义的 DeFi 创新实验，通过 Yearn Finance 我们看到去中心化的治理代币分发，充分激发了 DeFi 社区的挖矿和治理热情。

YFII 在 YFI 的基础上实现了 YIP-8 提案，探索了一种可能更公平的治理代币分发方案，并且短时间在社区内产生了较大影响，发展势头惊人。

## 安全建议

随着流动性挖矿和 DeFi 产品的火热，市面上涌现出来各种新型 DeFi 智能合约，组合性风险剧增。安比实验室提醒用户与任何 DeFi 项目交互时一定要注意安全第一，认清域名、合约地址，仔细审查所有与资金相关的操作，尽量不要与来源不明的智能合约交互。另外，我们应更多关注 DeFi 产品本身和智能合约安全，**分析价值基础和风险来源**，不盲信 APR，只投入能够承受损失的金额。

特别提醒，记得用本文中提供的线索**自行检查**参与的类 YFI 项目管理员权限。
