---
description: Answers to common Merkl questions to check before opening a support ticket.
---

# 📝 Merkl user FAQ

### What’s the frequency of rewards distribution?

The rewards distribution frequency varies by chain. On average, rewards are distributed approximately every 9 hours, but some chains — like Arbitrum — have distributions as frequently as every 4 hours. You can find the specific distribution frequency for each chain on this page: https://app.merkl.xyz/status

### I participated in a Merkl campaign but I can’t claim rewards on Merkl. Why?

If you're unable to claim rewards on Merkl after supplying liquidity to a pool, here are a few possible reasons:

- **Concentrated Liquidity Campaigns:** For campaigns based on concentrated liquidity, the Merkl engine requires at least one swap in the pool to calculate rewards accurately. If no swaps have occurred since the last engine run, the engine won’t process rewards yet, as one of the reward weights depends on swap fees.
    
    *Note:* Reward allocation is retroactive. Once a swap occurs, the Merkl engine will distribute all rewards accrued from the last engine run. If no engine run has happened since you added liquidity, you will receive all rewards accumulated from the time you initially supplied liquidity.
    
- **Campaign Status:** The campaign may not have started yet. Check its Status — if it’s marked as "Upcoming," it hasn’t started. Only "Live" campaigns are currently active, while completed ones are labeled "past."

### I performed an action (providing liquidity, lending, borrowing, …) on one chain, but rewards are distributed on another, is that normal?

Yes, the chain where you perform an action may differ from the one where rewards are distributed. For example, you might provide liquidity in a pool on the Base network and receive rewards on Optimism. This setup is defined by the incentivizers when they create the reward campaign.

You can find the chain where rewards are distributed in the campaign details on the opportunity page.

![Screenshot-FAQ.png](../.gitbook/assets/screenshot-FAQ.png)

### My wallet address has been blacklisted. Why?

Merkl automatically blacklists addresses that display suspicious behavior, such as wash trading to artificially inflate rewards. This usually involves addresses trading in pools where they hold positions specifically to maximize rewards.

If you believe you were blacklisted by mistake, please open a support ticket and provide your wallet address along with an explanation of your activity. Our team will review your case.

### The daily distribution of rewards didn't occur. Will I still receive my rewards?

Yes, rewards are retroactive. If there’s a delay in the Merkl engine run for a campaign, you’ll receive any missed rewards during the next engine run once the issue is resolved. Since the Merkl engine runs multiple times per day, you won’t have to wait long for missed rewards. For example, if a campaign didn’t distribute rewards today but the engine runs the following day after the issue is fixed, you’ll receive both the missed rewards and the current rewards.

One common reason for delays is the absence of a swap in the pool. Swaps are necessary to calculate certain rewards, so if no swap occurs, the campaign will be delayed until the next engine run after a swap takes place.

### How are APRs calculated?

The main APR displayed is calculated as:

$$
\frac{\text{Daily Rewards} \times 365}{\text{Adjusted TVL*}}

$$

* The adjusted TVL of a pool or market is:

- the Total TVL of the pool/market if none addresses are either backlisted or whitelisted
- the Total TVL minus the TVL of the blacklisted addresses if the campaign creator backlisted EOA addresses and/or smart contracts
- the Total TVL of the whitelisted contract (e.g., if a campaign is restricted to a Gamma vault, Adjusted TVL will include the vault’s TVL, including idle liquidity)