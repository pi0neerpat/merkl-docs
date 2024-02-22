---
description: Guide to integrate Merkl in your app for your incentives distribution
---

# 🔌 Integrate Merkl in your App

You can integrate Merkl pools in your app but you don't have to. All pools are listed on the [Merkl App](https://merkl.angle.money/), and users can claim their tokens from there. This doc will guide you through the different options to integrate Merkl in your app.

![Merkl front integration diagram](/.gitbook/assets/docs-merkl-front-integration.jpg)

Data about all Merkl incentivized pools across all supported chains can be found by calling our API endpoint: `https://api.angle.money/v2/merkl`.

When calling this payload, you must specify the `chainId` address. You may also specify a `user` address if you want additional information related to one user (like claimable amounts, liquidity in the pools, ...)

A typical query looks like: [`https://api.angle.money/v2/merkl?chainIds[]=10&user=0xfdA462548Ce04282f4B6D6619823a7C64Fdc0185`](https://api.angle.money/v2/merkl?chainIds[]=10&user=0xfdA462548Ce04282f4B6D6619823a7C64Fdc0185).

## Listing incentivized pools

When called for a specific chain, the API returns in a `pools` object a mapping between pool addresses of this chain (like Uniswap V3 pool addresses) and details on the APRs for providing liquidity.

Note that because how concentrated liquidity AMMs and the way the script works, two LPs with different positions may earn drastically different returns. As such, APRs displayed here are average measures of the returns LPs can earn. If you are to integrate Merkl in your app, you may want to build the APR indicators tailored to your use cases.

The `pools` object also has details about the past distributions for each incentivized pool: their start and end dates, the parameters set and the amount of incentives for each distribution.

You can filter from the `pools` object to only display the pools of your choice (for instance pools with tokens in a subset and with an active distribution).

## Displaying user rewards

The `rewardsPerToken` object lets you get how much rewards of a specific token can be claimed by a user for a specific pool.

This object maps each reward token to the total amount of rewards accumulated on the pool for a user. It includes the rewards that have already been claimed as well as to the amount of unclaimed tokens on the pool. It also details where these tokens have been earned (Uniswap V3, Gamma, ...).

## Claiming user rewards

On top of displaying how much has been earned by a specific address, the API can also be used to build the payload of a claim transaction for a user which accumulated rewards.

When called for a specific user, it returns in a `transactionData` payload with the list of `tokens` to claim, the `amounts`, and a Merkle `proof` required to claim each of them.

_Rewards are claimable per token: meaning that if you have accumulated rewards of several tokens, you may choose to only claim your rewards of one token type, but you may also choose to claim all your token rewards at once._

The contract on which rewards should be claimed is the `Distributor` contract which address can be found on [this docs](helpers.md#🧑‍💻-smart-contracts), or on the [Angle SDK](https://github.com/AngleProtocol/sdk).

You have two options to do that:

- Rely on Angle's API: we build the claim transaction payload for you and the associated proof, and you just call our API. This is the example shown below.
- Build the proof yourself and join it to the transaction data from the API. You can find a Github repository below showing how to do that.

{% hint style="info" %}
In any case, if a call is made to the correct `Distributor` contract and the `token` or `amount` doesn't match the `proof`, the transaction will revert.
{% endhint %}

Here is a script you may use to claim all the token rewards for a user on a chain.

```typescript
import {
  Distributor__factory,
  MerklAPIData,
  registry,
} from "@angleprotocol/sdk";
import { JsonRpcSigner } from "@ethersproject/providers";
import axios from "axios";

export const claim = async (chainId: number, signer: JsonRpcSigner) => {
  let data: MerklAPIData["transactionData"];
  try {
    data = (
      await axios.get(
        `https://api.angle.money/v2/merkl?chainIds[]=${chainId}&user=${signer._address}`,
        {
          timeout: 5000,
        }
      )
    ).data[chainId].transactionData;
  } catch {
    throw "Angle API not responding";
  }
  const tokens = Object.keys(data).filter((k) => data[k].proof !== undefined);
  const claims = tokens.map((t) => data[t].claim);
  const proofs = tokens.map((t) => data[t].proof);

  const contractAddress = registry(chainId)?.Merkl?.Distributor;
  if (!contractAddress) throw "Chain not supported";
  const contract = Distributor__factory.connect(contractAddress, signer);
  await (
    await contract.claim(
      tokens.map((t) => signer._address),
      tokens,
      claims,
      proofs as string[][]
    )
  ).wait();
};
```

{% hint style="info" %}
If you want to build the proof yourself, [this file](https://github.com/AngleProtocol/sdk/blob/288185227514ae5c5bb23f5d4b72680eb839f6cc/src/utils/merkl.ts#L15) will show you how to do that.
{% endhint %}

## Send rewards to pools

There is no need to use the API or the frontend specifically for this, but it's possible to directly reward pools on Merkl at the smart contract level.

The following script shows how to create one or multiple distributions at once on Merkl.

```typescript
import {
  ChainId,
  DistributionCreator__factory,
  Erc20__factory,
  registry,
} from "@angleprotocol/sdk";
import { parseEther } from "ethers/lib/utils";
import { ethers, web3 } from "hardhat";

async function main() {
  const [deployer] = await ethers.getSigners();
  const ZERO_ADDRESS = ethers.constants.AddressZero;
  const MAX_UINT256 = ethers.constants.MaxUint256;

  // Address of the reward token to sned
  const rewardTokenAddress = "0x84FB94595f9Aef81147cD4070a1564128A84bb7c";
  // Address of the pool
  const pool = "0x3fa147d6309abeb5c1316f7d8a7d8bd023e0cd80";
  // Chain on which distribution should be made
  const chainId = ChainId.OPTIMISM;

  const distributionCreatorAddress =
    registry(chainId)?.Merkl?.DistributionCreator!;
  const distributionCreator = DistributionCreator__factory.connect(
    distributionCreatorAddress,
    deployer
  );
  const rewardToken = Erc20__factory.connect(rewardTokenAddress, deployer);

  const params = {
    // Address of the pool to incentivize
    uniV3Pool: pool,
    // Address of the reward token (must be whitelisted)
    rewardToken: rewardToken.address,
    // Addresses to exclude from the distribution (or optionally addresses of the wrappers that are not automatically detected
    // by the script)
    positionWrappers: ["0xa29193Af0816D43cF44A3745755BF5f5e2f4F170"],
    // Type of the wrappers (3=blacklisted addresses)
    wrapperTypes: [3],
    // Amount of tokens to send for the WHOLE distribution
    amount: parseEther("350"),
    // Proportion of rewards that'll be split among LPs which brought token0 in the pool during the time
    // of the distribution
    propToken0: 4000,
    // Proportion of rewards that'll be split among LPs which brought token1 in the pool during the time
    // of the distribution
    propToken1: 2000,
    // Proportion of rewards that'll be split among LPs which accumulated fees during the time
    // of the distribution
    propFees: 4000,
    // Whether out of range liquidity should be incentivized
    isOutOfRangeIncentivized: 0,
    // Timestamp of the start of the distribution
    epochStart: 1676649600,
    // Number of hours for which the distribution should last once it has started
    numEpoch: 500,
    // Multiplier provided by the address boosting reward. In the case of a Curve distribution where veCRV
    // provides a 2.5x boost, this would be equal to 25000
    boostedReward: 0,
    // Address of the token which dictates who gets boosted rewards or not. This is optional
    // and if the zero address is given no boost will be taken into account
    boostingAddress: ZERO_ADDRESS,
    // These two parameters are useless when creating a distribution, you may specify here whatever you like
    rewardId: web3.utils.soliditySha3("europtimism") as string,
    additionalData: web3.utils.soliditySha3("europtimism") as string,
  };

  // Comment if you've already approved the contract with `rewardToken`
  console.log("Approving");
  await (
    await rewardToken
      .connect(deployer)
      .approve(distributionCreator.address, MAX_UINT256)
  ).wait();

  /*
  Before depositing a reward, you must make sure that:
  1. Your reward token is whitelisted for the reward distribution and that the per hour distribution amount
  will be higher than the limit
  2. You have read the T&C before signing them
  */

  console.log("Signing the T&C");
  const message = await distributionCreator.message();
  console.log(message);
  const signature = await deployer.signMessage(message);

  console.log("Depositing reward...");
  await (
    await distributionCreator
      .connect(deployer)
      .signAndCreateDistribution(params, signature)
  ).wait();
  console.log("...Deposited reward ✅");

  // Now if you want to create multiple distributions at once, you may also do it as well

  const params1 = {
    uniV3Pool: pool,
    rewardToken: rewardToken.address,
    positionWrappers: ["0xa29193Af0816D43cF44A3745755BF5f5e2f4F170"],
    wrapperTypes: [2],
    amount: parseEther("500"),
    propToken0: 4000,
    propToken1: 2000,
    propFees: 4000,
    isOutOfRangeIncentivized: 0,
    epochStart: 1680269613,
    numEpoch: 500,
    boostedReward: 0,
    boostingAddress: ZERO_ADDRESS,
    rewardId: "0x",
    additionalData: "0x",
  };

  const params2 = {
    uniV3Pool: pool,
    rewardToken: rewardToken.address,
    positionWrappers: ["0xa29193Af0816D43cF44A3745755BF5f5e2f4F170"],
    wrapperTypes: [2],
    amount: parseEther("750"),
    propToken0: 4000,
    propToken1: 2000,
    propFees: 4000,
    isOutOfRangeIncentivized: 0,
    epochStart: 1681565613,
    numEpoch: 500,
    boostedReward: 0,
    boostingAddress: ZERO_ADDRESS,
    rewardId: "0x",
    additionalData: "0x",
  };

  console.log("Depositing multiple rewards at once...");
  await (
    await distributionCreator
      .connect(deployer)
      .createDistributions([params1, params2])
  ).wait();
  console.log("...Deposited rewards ✅");
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```