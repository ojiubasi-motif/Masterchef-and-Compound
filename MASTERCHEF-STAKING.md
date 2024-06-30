# Masterchef Staking 
### Table of content
- [what is Masterchef Staking](#defination)
- [How masterchef works](#working)
- [Features of Masterchef Staking](#features)
- [Illustration with simulation](#example)
- [PoC](#poc)
- [Mathematical explanation](#math)
- [Conclusion](#conclusion)

## <font color="">What is Masterchef Staking?<a id="defination"></a></font>
MasterChef staking refers to a specific ```smart contract``` system commonly used in *decentralized finance (DeFi)* projects for managing ```staking pools``` and distributing rewards. The term “MasterChef” originated from the popular decentralized exchange and *yield farming* platform, **SushiSwap**, which employs a *MasterChef contract* to manage ```liquidity mining``` and ```staking``` activities.

## <font color="">How masterchef works<a id="working"></a></font>
1. Staking Tokens: Users *deposit* their tokens into a **MasterChef** *contract* to participate in *staking pools*.
2. Earning Rewards: The *contract* distributes *rewards* based on the amount of tokens *staked* and the *duration* of the *staking period*.
3. Claiming Rewards: Users can claim their earned rewards at any time, depending on the specific *contract* terms.
4. Reinvesting Rewards: Rewards can be reinvested back into the *staking pool* for compound growth.

## <font color="">Features of Masterchef Staking<a id="features"></a></font>
#### Multiple Pools:
- Diverse Opportunities: Users can choose from various staking pools, each with different reward structures and token pairs.
- Flexible Staking: Ability to stake multiple types of tokens within the same platform.

#### Governance:
- Community Involvement: Token holders can participate in governance decisions regarding the platform.
- Voting: Users can vote on proposals to adjust reward rates, add new pools, or make other changes.

#### Automated Rewards:
- Smart Contracts: Automatically distribute rewards based on predefined rules.
- Efficiency: Reduces the need for manual intervention and ensures fair distribution of rewards.

## <font color="">Illustration with simulation<a id="example"></a></font>
Let's assume that
```
RewardsPerBlock = $1
On block 0, Staker A deposits $100
On block 10, Staker B deposits $400
On block 15, Staker A harvests all rewards
On block 25, Staker B harvests all rewards
On block 30, both stakers harvests all rewards.
```
Staker A deposits $100 on block 0 and ten blocks later Staker B deposits $400.
For the first ten blocks, Staker A had 100% of their rewards, which is $10.
```
From block 0 to 10:
BlocksPassed: 10
BlockRewards = BlocksPassed * RewardsPerBlock 
BlockRewards = $10
StakerATokens: $100
TotalTokens: $100

StakerAShare = StakerATokens / TotalTokens
StakerAShare = 1
StakerAAccumulatedRewards = BlockRewards * StakerAShare
StakerAAccumulatedRewards = $10
```
On block 10, Staker B deposits $400.
Now on block 15 Staker A is harvesting its rewards.
While they got 100% rewards from blocks 0 to 10, from 10 to 15 they are only getting 20% (1/5)
```
From Block 10 to 15:
BlocksPassed: 5
BlockRewards = BlocksPassed * RewardsPerBlock 
BlockRewards = $5
StakerATokens: $100
StakerBTokens: $400
TotalTokens: $500

StakerAShare = StakerATokens / TotalTokens
StakerAShare = 1/5
StakerAAccumulatedRewards = (BlockRewards * StakerAShare) + StakerAAccumulatedRewards
StakerAAccumulatedRewards = $1 + $10

StakerBShare = StakerBTokens / TotalTokens
StakerBShare = 4/5
StakerBAccumulatedRewards = BlockRewards * StakerBShare
StakerBAccumulatedRewards = $4
```
Staker A harvests $11 and ```StakerAAccumulatedRewards``` resets to 0.
Staker B has accumulated $4 for these last 5 blocks.
Then 10 more blocks pass and B decides to harvest as well.
```
From Block 15 to 25:
BlocksPassed: 10
BlockRewards: $10
StakerATokens: $100
StakerBTokens: $400
TotalTokens: $500
StakerAAccumulatedRewards: $2
StakerBAccumulatedRewards: $8 + $4
```
Staker B harvests $12 and ```StakerBAccumulatedRewards``` resets to 0.
Finally, both staker harvest their rewards on block 30.
```
From Block 25 to 30:
BlocksPassed: 5
BlockRewards: $5
StakerATokens: $100
StakerBTokens: $400
TotalTokens: $500
StakerAAccumulatedRewards: $1 + $2
StakerBAccumulatedRewards: $4
```
Staker A harvests $3 and B harvests $4.
Staker has harvested in total $14 and B $16

## <font color="">Implementation via PoC<a id="poc"></a></font>
This way, for each action (*Deposit or Harvest*) we had to go through all *stakers* and calculate their *accumulated rewards*.

Here's a simple staking *contract* with this implementation:
```
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.4;

import "hardhat/console.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./RewardToken.sol";

contract StakingManager is Ownable{
    using SafeERC20 for IERC20; // Wrappers around ERC20 operations that throw on failure

    RewardToken public rewardToken; // Token to be payed as reward

    uint256 private rewardTokensPerBlock; // Number of reward tokens minted per block
    uint256 private constant STAKER_SHARE_PRECISION = 1e12; // A big number to perform mul and div operations

    // Staking user for a pool
    struct PoolStaker {
        uint256 amount; // The tokens quantity the user has staked.
        uint256 rewards; // The reward tokens quantity the user can harvest
        uint256 lastRewardedBlock; // Last block number the user had their rewards calculated
    }

    // Staking pool
    struct Pool {
        IERC20 stakeToken; // Token to be staked
        uint256 tokensStaked; // Total tokens staked
        address[] stakers; // Stakers in this pool
    }

    Pool[] public pools; // Staking pools

    // Mapping poolId => staker address => PoolStaker
    mapping(uint256 => mapping(address => PoolStaker)) public poolStakers;

    // Events
    event Deposit(address indexed user, uint256 indexed poolId, uint256 amount);
    event Withdraw(address indexed user, uint256 indexed poolId, uint256 amount);
    event HarvestRewards(address indexed user, uint256 indexed poolId, uint256 amount);
    event PoolCreated(uint256 poolId);

    // Constructor
    constructor(address _rewardTokenAddress, uint256 _rewardTokensPerBlock) {
        rewardToken = RewardToken(_rewardTokenAddress);
        rewardTokensPerBlock = _rewardTokensPerBlock;
    }

    /**
     * @dev Create a new staking pool
     */
    function createPool(IERC20 _stakeToken) external onlyOwner {
        Pool memory pool;
        pool.stakeToken =  _stakeToken;
        pools.push(pool);
        uint256 poolId = pools.length - 1;
        emit PoolCreated(poolId);
    }

    /**
     * @dev Add staker address to the pool stakers if it's not there already
     * We don't have to remove it because if it has amount 0 it won't affect rewards.
     * (but it might save gas in the long run)
     */
    function addStakerToPoolIfInexistent(uint256 _poolId, address depositingStaker) private {
        Pool storage pool = pools[_poolId];
        for (uint256 i; i < pool.stakers.length; i++) {
            address existingStaker = pool.stakers[i];
            if (existingStaker == depositingStaker) return;
        }
        pool.stakers.push(msg.sender);
    }

    /**
     * @dev Deposit tokens to an existing pool
     */
    function deposit(uint256 _poolId, uint256 _amount) external {
        require(_amount > 0, "Deposit amount can't be zero");
        Pool storage pool = pools[_poolId];
        PoolStaker storage staker = poolStakers[_poolId][msg.sender];

        // Update pool stakers
        updateStakersRewards(_poolId);
        addStakerToPoolIfInexistent(_poolId, msg.sender);

        // Update current staker
        staker.amount = staker.amount + _amount;
        staker.lastRewardedBlock = block.number;

        // Update pool
        pool.tokensStaked = pool.tokensStaked + _amount;

        // Deposit tokens
        emit Deposit(msg.sender, _poolId, _amount);
        pool.stakeToken.safeTransferFrom(
            address(msg.sender),
            address(this),
            _amount
        );
    }

    /**
     * @dev Withdraw all tokens from an existing pool
     */
    function withdraw(uint256 _poolId) external {
        Pool storage pool = pools[_poolId];
        PoolStaker storage staker = poolStakers[_poolId][msg.sender];
        uint256 amount = staker.amount;
        require(amount > 0, "Withdraw amount can't be zero");

        // Update pool stakers
        updateStakersRewards(_poolId);

        // Pay rewards
        harvestRewards(_poolId);

        // Update staker
        staker.amount = 0;

        // Update pool
        pool.tokensStaked = pool.tokensStaked - amount;

        // Withdraw tokens
        emit Withdraw(msg.sender, _poolId, amount);
        pool.stakeToken.safeTransfer(
            address(msg.sender),
            amount
        );
    }

    /**
     * @dev Harvest user rewards from a given pool id
     */
    function harvestRewards(uint256 _poolId) public {
        updateStakersRewards(_poolId);
        PoolStaker storage staker = poolStakers[_poolId][msg.sender];
        uint256 rewardsToHarvest = staker.rewards;
        staker.rewards = 0;
        emit HarvestRewards(msg.sender, _poolId, rewardsToHarvest);
        rewardToken.mint(msg.sender, rewardsToHarvest);
    }

    /**
     * @dev Loops over all stakers from a pool, updating their accumulated rewards according
     * to their participation in the pool.
     */
    function updateStakersRewards(uint256 _poolId) private {
        Pool storage pool = pools[_poolId];
        for (uint256 i; i < pool.stakers.length; i++) {
            address stakerAddress = pool.stakers[i];
            PoolStaker storage staker = poolStakers[_poolId][stakerAddress];
            if (staker.amount == 0) return;
            uint256 stakedAmount = staker.amount;
            uint256 stakerShare = (stakedAmount * STAKER_SHARE_PRECISION / pool.tokensStaked);
            uint256 blocksSinceLastReward = block.number - staker.lastRewardedBlock;
            uint256 rewards = (blocksSinceLastReward * rewardTokensPerBlock * stakerShare) / STAKER_SHARE_PRECISION;
            staker.lastRewardedBlock = block.number;
            staker.rewards = staker.rewards + rewards;
        }
    }
}
```

The ```updateStakersRewards``` is responsible to loop over all staker and update their accumulated rewards every time someone deposits, withdraws or harvests their earnings.

But what if we could avoid this loop?

## <font color="">Applying Mathematics<a id="math"></a></font>
If we see Staker A rewards **as** a **sum** of their rewards on each group of blocks
```
StakerARewards = 
StakerA0to10Rewards + 
StakerA10to15Rewards + 
StakerA15to25Rewards + 
StakerA25to30Rewards
```
And if we see their rewards from the block N to M **as** the multiplication between the rewards that were distributed in the same range **by** their *share* in the same range
```
StakerANtoMRewards = BlockRewardsOnNtoM * StakerAShareOnNtoM
```
Then we get the staker rewards as the sum of the multiplication between the rewards and their share for each range up to the end
```
StakerARewards = 
(BlockRewardsOn0to10 * StakerAShareOn0to10) + 
(BlockRewardsOn10to15 * StakerAShareOn10to15) + 
(BlockRewardsOn15to25 * StakerAShareOn15to25) + 
(BlockRewardsOn25to30 * StakerAShareOn25to30)
```
And using the following formula that represents the staker share as their tokens divided by the total tokens in the pool
```
StakerAShareOnNtoM = StakerATokensOnNtoM / TotalTokensOnNtoM
```
We have this
```
StakerARewards = 
(BlockRewardsOn0to10 * StakerATokensOn0to10 / TotalTokensOn0to10) + 
(BlockRewardsOn10to15 * StakerATokensOn10to15 / TotalTokensOn10to15) + 
(BlockRewardsOn15to25 * StakerATokensOn15to25 / TotalTokensOn15to25) + 
(BlockRewardsOn25to30 * StakerATokensOn25to30 / TotalTokensOn25to30)
```
But, in this case, the staker had the same amount of tokens deposited at all ranges
```
StakerATokensOn0to10 = 
StakerATokensOn10to15 = 
StakerATokensOn15to25 = 
StakerATokensOn25to30 = 
StakerATokens
```
Then we can simplify our ```StakerARewards``` formula
```
StakerARewards = 
(BlockRewardsOn0to10 * StakerATokens / TotalTokensOn0to10) + 
(BlockRewardsOn10to15 * StakerATokens / TotalTokensOn10to15) + 
(BlockRewardsOn15to25 * StakerATokens / TotalTokensOn15to25) + 
(BlockRewardsOn25to30 * StakerATokens / TotalTokensOn25to30)
```
And by putting ```StakerATokens``` on evidence we have this
```
StakerARewards = StakerATokens * (
  (BlockRewardsOn0to10 / TotalTokensOn0to10) + 
  (BlockRewardsOn10to15 / TotalTokensOn10to15) + 
  (BlockRewardsOn15to25 / TotalTokensOn15to25) + 
  (BlockRewardsOn25to30 / TotalTokensOn25to30)
)
```
We can make sure that it works with our scenario by replacing these big words with numbers and getting the total rewards for Staker A
```
StakerARewards = 100 * (
  (10 / 100) + 
  (5  / 500) + 
  (10 / 500) + 
  (5  / 500)
)
```
```
StakerARewards = 14
```
Which matches with what we were expecting

Let's do the same for staker B
```
StakerBRewards = 
(BlockRewardsOn10to15 * StakerBTokens / TotalTokensOn10to15) + 
(BlockRewardsOn15to25 * StakerBTokens / TotalTokensOn15to25) + 
(BlockRewardsOn25to30 * StakerBTokens / TotalTokensOn25to30)
```
```
StakerBRewards = StakerBTokens * (
  (BlockRewardsOn10to15 / TotalTokensOn10to15) + 
  (BlockRewardsOn15to25 / TotalTokensOn15to25) + 
  (BlockRewardsOn25to30 / TotalTokensOn25to30)
)
```
```
StakerBRewards = 400 * (
  (5  / 500) + 
  (10 / 500) + 
  (5  / 500)
)
```
```
StakerBRewards = 16
```
Now that both stakers rewards are matching with what we've seen before, let's check what we can reuse in both rewards calculation.

As you can see, both stakers rewards formulas have a common sum of divisions

```
(5 / 500) + (10 / 500) + (5 / 500)
```
The **SushiSwap**'s contract call this sum ```accSushiPerShare```, so let's call each division as ```RewardsPerShare```
```
RewardsPerShareOn0to10  = (10 / 100)
RewardsPerShareOn10to15 = (5  / 500)
RewardsPerShareOn15to25 = (10 / 500)
RewardsPerShareOn25to30 = (5  / 500)
```
And instead of ```accSushiPerShare``` we will call their sum ```AccumulatedRewardsPerShare```
```
AccumulatedRewardsPerShare = 
RewardsPerShareOn0to10 + 
RewardsPerShareOn10to15 + 
RewardsPerShareOn15to25 + 
RewardsPerShareOn25to30
```
Then we can say that ```StakerARewards``` is the multiplcation of ```StakerATokens``` by ```AccumulatedRewardsPerShare```

```
StakerARewards = StakerATokens * 
AccumulatedRewardsPerShare
```
Since ```AccumulatedRewardsPerShare``` is the same for all stakers, we can say that ```StakerBRewards``` is that value minus the rewards they didn't get from blocks 0to10
```
StakerBRewards = StakerBTokens * 
(AccumulatedRewardsPerShare - RewardsPerShareOn0to10)
```
This is important, because even though we can use ```AccumulatedRewardsPerShare``` for every staker rewards calculation, we have to subtract the ```RewardsPerShare``` that happened before their *Deposit/Harvest* action.

Let's find out how much the Staker A has *harvested* on their first *harvest* using what we discovered out so far.

#### Finding out rewardDebt
We know that the rewards that Staker A got is the sum of their first and last harvest, that is from blocks 0to15 and 15to30.
Also, we know that we can get the same value with the ```StakerARewards``` formula we just used above
```
StakerARewards = StakerARewardsOn0to15 + StakerARewardsOn15to30
StakerARewards = StakerATokens * AccumulatedRewardsPerShare
```
If we isolate ```StakerARewardsOn15to30``` in the first formula and replace its ```StakerATokens``` with the second one
```
StakerARewardsOn15to30 = StakerARewards - StakerARewardsOn0to15
StakerARewards = StakerATokens * AccumulatedRewardsPerShare
```
we get
```
StakerARewardsOn15to30 = StakerATokens * 
AccumulatedRewardsPerShare - StakerARewardsOn0to15
```
Now we can use the following formula for blocks 0to15
```
StakerARewardsOn0to15 = StakerATokens * 
AccumulatedRewardsPerShareOn0to15
```
And replace StakerARewardsOn0to15 in the previous one
```
StakerARewardsOn15to30 = 
StakerATokens * AccumulatedRewardsPerShare -
StakerATokens * AccumulatedRewardsPerShareOn0to15
```
Now you might have noticed that we can isolate StakerATokens again
```
StakerARewardsOn15to30 = StakerATokens * 
(AccumulatedRewardsPerShare - AccumulatedRewardsPerShareOn0to15)
```
And that's very similar to the formula we got for StakerBRewards previously
```
StakerBRewards = StakerBTokens * 
(AccumulatedRewardsPerShare - RewardsPerShareOn0to10)
```
We can also replace some values to check if it actually works
```
StakerATokens = 100
AccumulatedRewardsPerShare = (10 / 100) + (5 / 500) + (10 / 500) + (5 / 500)
AccumulatedRewardsPerShare = (10 / 100) + (5 / 500)
StakerARewardsOn15to30 = StakerATokens * 
(AccumulatedRewardsPerShare - AccumulatedRewardsPerShareOn0to15)
StakerARewardsOn15to30 = 100 * ((10 / 500) + (5 / 500))
StakerARewardsOn15to30 = 3
```
So yeah, it works.

This means that if we save the ```AccumulatedRewardsPerShare``` value multiplied by the staker tokens amount each time their deposits or withdraws we can use this value to simply subtract it from their total rewards.

This is called ```rewardDebt``` on the MasterChef's contract.

It's like calculating a staker total rewards since block 0, but removing the rewards they already harvested or the rewards their were not eligibly to claim because they weren't staking yet.

<!-- The solution is implemented as shown in [this code base]() -->
### The ```AccumulatedRewardsPerShare``` implementation
Using the previous contract as base, we can simply calculate ```accumulatedRewardsPerShare``` on ```updatePoolRewards``` function (renamed from ```updateStakersRewards```) and get the staker ```rewardsDebt``` each time they perform an action.

You can see the diff code on [this code base](https://github.com/ojiubasi-motif/Masterchef-and-Compound/blob/master/masterchef.sol).

### Gas Saving
The reason we are avoiding a loop is mainly to save gas. As you can imagine, the more stakers we have, the more expensive the ```updateStakersRewards``` function gets. 

## <font color="">Conclusion and Reference<a id="conclusion"></a></font>
MasterChef staking provides a robust framework for managing staking and liquidity mining activities in the DeFi space. By utilizing smart contracts, it ensures efficient and automated distribution of rewards, offering users a flexible and community-driven staking experience. As DeFi continues to grow, MasterChef staking will likely remain a key component of many decentralized platforms, enabling users to maximize their returns while contributing to the security and functionality of blockchain networks.

Many protocols adopt the masterchef staking method like [Synthetix](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol), Hovever there few differences on how Synthetix chose to implement theirs, which makes their(Synthetix) protocol somewhat less efficient compared to **Sushi**'s masterchef contract. below are some of the differences:
### Differences between Synthetix and MasterChef

Synthetix and MasterChef both use the same mechanism to accumulate the reward per token based on amount staked. The major difference is that rather than tracking reward debt, Synthetix stores a snapshot of the reward accumulator when the user last interacted with the contract. The difference between the current reward accumulator and the snapshot is used to calculate the rewards to the user’s account.


That difference is added to a per-user rewards mapping and accumulates there until the user calls getRewards(). This extra bookkeeping makes the Synthetix algorithm less efficient.

The rest of the differences are fairly minor

- MasterChef has ```deposit()``` and ```withdraw()```. 
Synthetix has ```stake()```, ```withdraw()```, and ```getReward()```.
- MasterChef uses ```blocks``` as a unit of time. 
Synthetix uses the timestamp.
- MasterChef ```mints``` rewards to itself as described in the above sections. 
Synthetix assumes the admin has already transferred the rewards to the contract and does not mint rewards.
- MasterChef distributes rewards from a configurable ```startBlock``` to ```lastRewardBlock```. 
Synthetix is hardcoded to distribute rewards over the course of a week after the admin starts the ```clock```. Synthetix will not necessarily distribute the entire balance of rewards in the contract, but an amount specified by the admin.
- MasterChef transfers the reward to the user whenever they call ```deposit()``` or ```withdraw()``` with non-zero amounts. 
Synthetix accumulates the reward due to the user in a mapping called rewards but does not transfer it to the user until they explicitly call ```getRewards()```.
- MasterChef supports multiple pools within the same contract, and divides up the rewards by the weight of the pool.
Synthetix only has one pool

[you can consult the code for SushiSwap MasterChef Staking.](https://github.com/sushiswap/sushiswap/blob/271458b558afa6fdfd3e46b8eef5ee6618b60f9d/contracts/MasterChef.sol)
