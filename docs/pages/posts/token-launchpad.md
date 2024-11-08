---
layout: page
title: "Token Launchpad on Base"
permalink: /posts/token-launchpad
---

In 2024, I was a core contributor to a token launchpad on the Base blockchain. As the lead backend engineer, I developed the smart contracts and built the backend infrastructure that thousands of users interacted with.

A token launchpad allows users to create an ERC20 token, mint a designated supply, and allocate it to a bonding curve. By depositing Ethereum into this curve, buyers can acquire tokens. Once all tokens on the bonding curve are sold, the accrued ETH and a set token reserve are deposited into a decentralized exchange (DEX) like Uniswap, enabling normal trading. By burning the LP tokens created when depositing the funds on Uniswap, we prevent the liquidity from being withdrawn, ensuring it remains in the pool.

Here are some reasons creators and users might prefer using a launchpad over managing the process manually:

- **For creators, simplicity** — No prior knowledge is required for token creation or DEX interaction.
- **For creators and users, price discovery** — The bonding curve offers a pricing mechanism, even when the token has few or no holders.
- **For users, security** — Once deposited in the DEX, liquidity is non-retractable, reducing the likelihood of a "rug pull" by the creator.
- **For users, guaranteed liquidity** — Buyers on the bonding curve are assured liquidity if they decide to sell back into the curve.

## Differentiating Factors

Several features made our launchpad stand out from others:

- **Verified, Public, and Audited Protocol** — Our protocol's smart contract was fully verified, publicly accessible, and audited. In contrast, most other launchpads used unverified code, preventing users from reviewing it.
- **Engagement System for Token Visibility** — Users earned points with each trade, which they could use to vote for a “Coin of the Day.” The selected token would gain prominent visibility across the platform for a day.
- **On-Chain Referral System** — Our launchpad offered a referral code system, allowing users to include a referral code in transactions. Referrers received a portion of the fees, incentivizing community growth.
- **Direct Uniswap Trading** — Once a token transitioned off the bonding curve, users could seamlessly trade it on Uniswap directly through the launchpad.

## Design of the Launchpad

![Design of the Launchpad](/assets/token-launchpad.svg)

The main components of the system are outlined above. Below, each component is described in more detail, along with how they interact:

1. **Protocol Contracts**
    - Core smart contracts for the launchpad protocol, facilitating token creation, bonding curve trading, and promotion to Uniswap v2 pools.
    - Detailed further in the next section.
2. **Frontend**
    - Key user actions:
        - Create a new ERC20 token with a fixed supply on a bonding curve.
        - Browse and trade tokens on the bonding curve or promoted tokens on Uniswap.
        - View holder distributions, user profiles by address, and token allocations.
        - Engage in global and per-token chat threads.
        - Vote for tokens to compete for “Coin of the Day,” using points earned through trades.
3. **CRUD API**
    - Powers frontend data by accessing token, trade, and user data from the database.
    - Manages off-chain actions like voting, chatting, etc., sending real-time updates to a websocket server to keep users current.
4. **Blockchain Listener**
    - Constantly observes the protocol contracts for new events (e.g., trades, token creations, promotions to Uniswap).
    - Updates relevant token states in the database and forwards these events to the websocket server for real-time frontend updates.
5. **Websocket Server**
    - Delivers real-time frontend updates for trades, token creations, voting, daily races, and chat interactions, fostering live engagement.
6. **Mentions Listener**
    - Monitors Twitter/X and Farcaster for posts mentioning a sentinel account with the intent of creating a new token on the platform.
    - When posts meet specified criteria (e.g., image, name, description, ticker), the listener triggers the creation of a token via the protocol contract.
7. **Cross-Launchpad Activity Listener**
    - Tracks tokens on other launchpads to stimulate activity and exposure during the launchpad's early stages.
    - Provides direct links to other launchpads for token trading.
8. **Uniswap Trade Component**
    - Abstraction layer for trading directly on Uniswap through the launchpad, using the Uniswap Universal Router for v2 pools with optional fees.
    - Detailed further in the next section.

## The Protocol Contracts

### The Bonding Curve

At the core of the contract was the bonding curve, which determined token pricing at any given point. The bonding curve used was linear, represented by the equation `y = m * x + b`, where:

- **x** — Total supply of tokens sold
- **y** — Current price of the token being sold
- **m** — Slope of the bonding curve
- **b** — Starting price of the token sale

From this formula, we derived key equations:

- **Token Price** — `y = m * x + b`
    - Calculates the current price of the token being sold.
- **Total Liquidity** — `total_liquidity = b * total_tradeable_supply + (m * total_tradeable_supply^2) / 2`
    - Represents the total ETH generated if all tokens are sold. This formula is based on the integral of the linear price equation, representing the area under the curve, which gives the total ETH liquidity.
- **ETH Value for Trade** — `eth_value_for_trade = ((m * amount_traded^2) / 2) + m * x * amount_traded + b * amount_traded`
    - Calculates the amount of ETH required for a specific trade.
        - For a **buy** transaction, `amount_traded` is positive, yielding the ETH needed to purchase the tokens.
        - For a **sell** transaction, `amount_traded` is negative, indicating the ETH generated from selling those tokens.

### Promoting to Uniswap

Once all tokens had been sold or the bonding curve for a given token was fully funded with ETH, the token would be promoted to Uniswap. During this promotion, there were two possible scenarios.

In the first scenario, no Uniswap v2 pool existed for the token. Here, the contract would handle the creation of the pool, funding it with the pre-set reserve tokens along with all the liquidity accumulated on the bonding curve. This setup ensured the initial price on Uniswap matched the final price of the bonding curve.

In the second scenario, a Uniswap v2 pool for the token already existed, created by an external actor. In this case, while the actor couldn’t add tokens due to transfer restrictions until promotion, they could deposit ETH into the pool. This posed a risk by potentially altering the ETH-to-token ratio, allowing the actor to manipulate the token's starting price. To prevent such exploitation, the contract checked the ETH amount in the existing pool and minted additional reserve tokens as needed to maintain the expected ETH-to-token ratio, ensuring the starting price accurately reflected the bonding curve’s final price.

Although the process was relatively straightforward, implementing these safeguards was crucial to avoid potential exploits.

## The Uniswap Trade Component

This component was especially interesting because it allowed us to take fees in ETH while trading on a Uniswap v2 pool—something that’s normally not possible when interacting directly with v2 pool contracts. However, the Uniswap Universal Router made it possible.

The Universal Router enables more complex trade interactions by using a set of encoded commands. Here’s a breakdown of how it works, but you can check the [GitHub repo here](https://github.com/Uniswap/universal-router) for more details.

We supported four main types of trades:

- Exact Input, ETH → ERC20
- Exact Input, ERC20 → ETH
- Exact Output, ETH → ERC20
- Exact Output, ERC20 → ETH

An “Exact Input” trade specifies the exact amount of input currency, with the output amount calculated based on that input. An “Exact Output” trade specifies the exact output amount, and the required input is calculated.

For example, the exact input ETH → ERC20 trade command flow works like this:

1. **Transfer Fee (optional)** — The `TRANSFER` command sends a portion of ETH as a fee to the fee wallet if required.
2. **Wrap ETH** — The `WRAP_ETH` command sends ETH to the router for holding during the swap.
3. **V2 Swap** — The `V2_SWAP_EXACT_IN` command sets the exact amount of ETH to swap and specifies the minimum ERC20 output, accounting for slippage.
4. **Unwrap WETH** — If the ERC20 isn’t held in the router, `UNWRAP_WETH` transfers the resulting ETH directly to the user.
5. **Transfer Remaining ETH** — Any leftover ETH is refunded to the user.

## Final Thoughts

I hope this post gave you some insight into what a token launchpad is and how it is built. Thanks for reading! 
