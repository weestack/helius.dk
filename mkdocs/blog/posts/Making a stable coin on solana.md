---
date:
  created: 2025-02-17
authors:
  - weestack
comments: true
blog_toc: true
categories:
  - Solana Smart contract
  - Solana stable coin
tags:
    - Solana
---

# Making a stable coin on Solana


Recently, I spoke with someone about creating a stablecoin. I had given it some thought myself and asked him:
> So how do you stabilize it on the curve?

By this, I meant how to keep it steady at $1 or €1 because whenever you mint a coin using the token program and start swapping in and out, the price fluctuates.  
He responded:
>  You need to make triggers to keep the peg in place

He also mentioned collateralization and other concepts, but that’s not particularly relevant to this article.  
Being me, I kept thinking about it, and these are my (probably naive) solutions to creating a stablecoin:  

## Idea one Constant product
![Swap distribution on a stable coin](/static/stable_coin_on_solana/Screenshot 2025-02-17 at 21.11.58.png)
/// caption
Swap distribution on a stable coin
///

### The math
When a token is created on Solana using the token program, its value is determined by the [Constant product](https://spl.solana.com/token-swap#constant-product). Below is a simplified version of the necessary calculation:
Minted token supply / Solana supply
5000 / 5000 = 1 sol

If we wanted to create a token that always remains worth 1 SOL, we could develop a smart contract that mints on buys and burns on sells to maintain equilibrium.
  
However, a stablecoin should probably track something more stable—like the Euro. For that, we need an oracle.
> An [oracle](https://docs.chain.link/docs/solana/data-feeds-solana/) in smart contracts is a service to fetch off-chain data, such as the euro price.

Now, the math effectively becomes:  
minted_supply / (solana supply / euro price)
5000 / (5000 / 7.5) =
5000 / 666.6667 =
7.5  

The status quo can still be maintained by minting and burning with corresponding buys and sells directly within the smart contract. With the help of the oracle, it can automatically adjust itself whenever the Euro fluctuates.

That concludes Idea One, which already seems to exist in the form of this [stable swap program](https://github.com/saber-hq/stable-swap/blob/master/stable-swap-math/src/curve.rs#L225).

## Idea two, liquidity providing
*Idea two* builds upon the first but introduces a slight... let’s say profit incentive (Disclaimer: This is just an idea—there may be regulatory concerns).  

Let’s say you create a €100M stablecoin, meaning you swap €100M for the equivalent amount of SOL. At today’s exchange rate (€169.49 per SOL), that would equal 590,005.31 SOL.  
To maintain its value at €1, we apply the formula from Idea One and isolate the unknown.  

The real difference in this idea is using event listeners on Solana for our mint and taking the following actions:  

- If the value drops below €1, we swap in SOL to push it back up.  
- If the value rises above €1, we swap out SOL to bring it back down.  

This is liquidity providing in its purest/simplest form. However, there are several challenges to consider, such as sandwich attacks, arbitrage bots, and so on.

This being liquidity providing also means you need to account for price fluctuations while also ensuring the ability to stake the invested supply.

- Sandwich attacks can be mitigated by using Jito to wrap swaps.
- Arbitrage bots are harder to counter and require technical finesse—primarily by being faster.

## Wrap up
That’s it for my thoughts on stablecoins. They are by no means a new concept and operate in a heavily regulated field. This article may not have provided the exact insights you were hoping for, but at the very least, it should give you a few ideas to explore on this topic. 

## Disclaimer:
The information provided in this blog is for educational and informational purposes only. It does not constitute financial, legal, or investment advice. Always check the regulations in effect in your jurisdiction before considering any concepts discussed here. I take no responsibility for any actions taken based on this content.
