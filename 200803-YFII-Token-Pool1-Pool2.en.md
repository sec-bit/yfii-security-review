# Security Review on YFII Liquidity Mining Smart Contracts

*Thanks to community volunteers Alex Vane and Terence C. for the translation!*

> https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.md
> 
> **Available in [ [English](https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.en.md) | [中文](https://github.com/sec-bit/yfii-security-review/blob/master/200803-YFII-Token-Pool1-Pool2.md) ]**


[YFII](https://yfii.finance/) is a newly-initiated DeFi mining pool. At the invitation of community, [SECBIT Labs](https://secbit.io) conducted a security study of YFII smart contracts from July 27 to August 2, 2020.

The analysis was performed on the following contracts.

- YFII Pool 1: 0xb81D3cB2708530ea990a287142b82D058725C092
- YFII Pool 2: 0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a
- YFII Token: 0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83
- BPT Token: 0x16cAC1403377978644e78769Daa49d8f6B6CF565

Preliminary analysis shows that the above four contracts do not contain critical security vulnerabilities. SECBIT hopes to make a summary of the research process through this article. However, token prices, economic models, other external contract modules, and new contracts in the future are not included in this discussion.

## What are YFII and YFI

Yearn Finance is a DeFi yield aggregator that introduced its governance token YFI on July 17, rapidly gained popularity in the liquidity mining space, due to its novel distribution mechanism and governance model. YFII is a fork of YFI, with the implementation of the YIP-8 (A proposal for improvement regarding token distribution schedule to YFI), a halving mechanism similar to Bitcoin.

## What Changes YFII Made Relative to YFI

For the moment, YFII contract codes directly fork from Yearn Finance, with minor changes to support for halving mechanism on a regular basis.

The following table shows the involved contracts, as well as the correspondence and addresses of YFI contracts.

|      Contract      |                        YFII Corresponding Address                         |                YFI Corresponding Address                |
| :------------: | :----------------------------------------------------------: | :----------------------------------------: |
|     Pool1 (YearnRewards)      | [0xb81D3cB2708530ea990a287142b82D058725C092](https://etherscan.io/address/0xb81D3cB2708530ea990a287142b82D058725C092#code) | [0x0001FB050Fe7312791bF6475b96569D83F695C9f](https://etherscan.io/address/0x0001FB050Fe7312791bF6475b96569D83F695C9f#code) |
|     Pool2 (YearnRewards)     |          [0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a](https://etherscan.io/address/0xAFfcD3D45cEF58B1DfA773463824c6F6bB0Dc13a#code)          | [0x033E52f513F9B98e129381c6708F9faA2DEE5db5](https://etherscan.io/address/0x033E52f513F9B98e129381c6708F9faA2DEE5db5#code) |
| YFI/YFII Token (ERC20) |          [0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83](https://etherscan.io/address/0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83#code)          | [0x0bc529c00C6401aEF6D220BE8C6Ea1667F6Ad93e](https://etherscan.io/address/0x0bc529c00C6401aEF6D220BE8C6Ea1667F6Ad93e#code) |
|   BPT Token (Balancer)   |          [0x16cAC1403377978644e78769Daa49d8f6B6CF565](https://etherscan.io/address/0x16cAC1403377978644e78769Daa49d8f6B6CF565#code)          | [0x95c4b6c7cff608c0ca048df8b81a484aa377172b](https://etherscan.io/address/0x95c4b6c7cff608c0ca048df8b81a484aa377172b#code) |

More specifically, YFI/YFII are both standard ERC-20 tokens with the same implementation, which also implement the same minting and simple governance logic, to govern token contracts.

The BPT token is a Balancer Pool token contract, a liquidity token for market makers, which is contractually created by Balancer's BFactory contract. The contract code was previously [audited by Trail of Bits and Consensys Diligence](https://docs.balancer.finance/protocol/security/audits).

Pool 1 and Pool 2 are liquidity mining contracts for the distribution of the governance token, with consistent code implementation known as YearnRewards contracts, including the changes of YFII relative to YFI.

The YFII Pool1 and Pool2 contracts have two new modifier functions relative to the original code, `checkStart()` and `checkhalve()`, which are used to control the starting time of mining and the periodic halving of the governance token.

## Core Contracts of YFII and YFI in a Nutshell

The core contract code for YFII and YFI liquidity mining, YearnRewards, is derived from the [Unipool](https://github.com/Synthetixio/Unipool/blob/master/contracts/Unipool.sol) of the Synthetix project. It was initially used to reward market makers with SNX token for providing liquidity to ETH/sETH pairs on Uniswap. The code has been previously [audited by Sigma Prime](https://github.com/sigp/public-audits/blob/master/synthetix/unipool/review.pdf).

YearnRewards-based liquidity mining procedure can be described as follows:

1. The admin account with `RewardDistribution` privileges sets the reward amount in advance by calling the `notifyRewardAmount()` function of the YearnRewards contract. The YFI token for the corresponding amount shall be transferred from the YFI minter into YearnRewards contracts.
2. The miner, who provides liquidity to target DeFi contracts designated by YearnRewards contract, can receive corresponding liquidity tokens, also commonly known as Pool Token, which can be used to exchange assets back and earn interest or fee income.
3. The miner deposits the resulting Pool Token into the YearnRewards contract by calling the `stake()` function. The contract automatically calculates the miner's entitlement based on the length of the stake and the size of the miner's deposit as a percentage of the total pool size.
4. At any time, the miner can withdraw his or her due rewards (YFI Token) and any previously deposited Pool Tokens.

Usually, one YearnRewards contract is dedicated to liquidity mining for a single specific DeFi project, such as Pool1 to the y pool on the Curve project and Pool2 to the YFI-DAI pool on Balancer protocol.

## Some of the Findings

The changes to YFII compared to YFI, as mentioned above, are minor overall.

Two new modifier functions have been added to constrain three main functional fuctions including `stake()`, `withdraw()`, and `getReward()`.

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

A new line of code has been added to the `notifyRewardAmount()` function to directly control the YFI token contract to mint the specified number of tokens to the current YearnRewards contract be used as a reward for distribution. Therefore, the Pool1 and Pool2 contracts must be minter of the YFII token contract.

It makes YFII slightly different from YFI in token distribution details logically. For YFI, every period of the YFI award will need to be set by a specific address and transferred with tokens. Whereas YFII only performs a `notifyRewardAmount()` operation before the first period begins, after which the yield is automatically regularly halved as the users invoking with the smart contract.

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

Besides, we discussed code details with community developers Madao and gaojin. And Madao mentioned that the execution of the automatic halving of token's yield relies on the execution of the `checkhalve()` function. It relies on user interaction with the contract further. The execution time cannot be precisely controlled to be the end of the previous cycle. There will be a time difference between the halving time and the expected time. The actual halving of the contract will most likely occur later than expected.

In particular, the time difference between the two cycles will be taken into account when the smart contract is calculating the reward. It will result in the reward value calculated for each user being slightly higher than the expected value, and a specific error occurs. As a result, we find that as long as the error exists, the last person to withdraw the reward from the pool may theoretically not be able to withdraw it properly. The contract is halved at the same time as minting YFII token to the pool contract. Due to the existence of the previous error, the book income of the user is higher than the token amount mints out. The magnitude of the error is calculated by multiplying the time difference `Delta` by the reward rate after the halving.  The `Delta` is the time difference between the end of each cycle and when the next halving occurs.

![](https://i.imgur.com/JGULj1o.png)

According to the above graph, we take the average delay time for halving as 60 seconds, and the cumulative error is within 1 YFII. As long as the time error is small enough, plus the next cycle's continuous token as a supplement, the error problem has little effect.

[@ThinkingETH](https://twitter.com/ThinkingETH/status/1290633339158687746) reminded that `withdraw()` should not have the `checkhalve()` modifier. If the contract is accidentally removed as a minter, all user funds may be frozen after the halving since the contract function will fail. 

Luckily, YFII has set the token's owner to zero address (see the next section for details). The Pool1 and Pool2 should always be the minter of the YFII token. So this risk does not currently exist for YFII.  However, as smart contract code, this should be implemented more rigorously and robustly, otherwise many users' money could be at high risk. In particular, many projects that go further with forking YFII code can be tragic if developers don't understand the details. [@DoveyWan](https://twitter.com/DoveyWan/status/1290541716290670592) and [@oli_vdb](https://twitter.com/oli_vdb/status/1290370855709573122) also mentions a similar security incident.

## YFII Administrator Rights Processing

YFI-class tokens have mint interfaces, while addresses with mint privileges can issue additional tokens. Besides, YFI-class tokens also have a governance administrator with permission to add and remove minters. Ideally, these addresses with special rights should be multi-signature contracts or other specialized contracts.

YearnRewards contracts also have `rewardDistribution` privilege addresses, which are used to call the `notifyRewardAmount()` function to set the reward amount. YearnRewards contracts also have `owner` privilege addresses that are used to set the `rewardDistribution` address.

The current practice of the YFII project is to set the `governance` administrator address of YFII token, plus the `rewardDistribution` and `owner` addresses of Pool1 and Pool2 to 0 addresses. The administrator privilege destruction transaction can be found at https://burn.yfii.finance/. After inspection, we confirmed that administrator privileges are indeed destroyed.

Only the Pool1 and Pool2 contract addresses currently have mint privileges on the YFII Token, which are necessary to achieve the periodic halving and cannot be abused in the future.

It is worth mentioning that the original YFI Token code implementation does not add an Event to the `addMinter()` privileged function, which is a bad practice, making it impossible for ordinary users to check ou how many minters in the contract quickly. Worryingly, it's effortless for YFI-like projects to have terrible back doors.

The YFII Token contract has [only two `addMinter() ` records]((https://api.blockchair.com/ethereum/transactions?q=input_hex(^983b2d56),recipient(0xa1d0E215a23d7030842FC67cE582a6aFa3CCaB83))), adding mint permissions to the Pool1 and Pool2 contracts, respectively, **without introducing any redundant minters**.

## Summary

YFI is a significant DeFi experiment. By Yearn Finance, we have seen the decentralized distribution of governance tokens, which fully stimulate the mining and governance enthusiasm of the DeFi community.

Based on YFI, YFII has accomplished the YIP-8 proposal, exploring a more equitable governance token distribution scheme and a great impact and amazing momentum of development in the community, in a short time.

## Security Recommendations

With the popularity of liquidity mining and DeFi products, various DeFi smart contracts have emerged on the market, while combinatorial risk has increased sharply. We remind that when users interact with any DeFi project, they must pay attention to security first, by identifying domains, contract addresses, and carefully reviewing all fund-related operations to avoid interacting with unknown smart contracts.

Moreover, we should pay more attention to the DeFi product itself and smart contract security, and analyze the value basis and risks, only investing the amount one can afford to lose, instead of blindly chasing high yield.

For a special reminder, do not forget to use the tips provided in this article to check for administrator permissions of similar YFI projects by yourself.

---

Translated by Alex Vane, Terence C., and SECBIT Labs.
