---
layout: page
title: "Token Launchpad on Base"
permalink: /posts/token-launchpad
---

In 2024 I was a core contributor to a token launchpad on the Base blockchain. As the lead backend engineer I developed the smart contracts and built the backend infrastructure that thousands of users interacted with.

A token launchpad's smart contracts allow users to create a token, mint a designated supply, and allocate it to a bonding curve. By depositing Ethereum into this curve buyers can acquire tokens. Once all tokens on the bonding curve are sold the accrued ETH and a set token reserve are deposited into a decentralized exchange (DEX) like Uniswap for normal trading. Since the contract burns the liquidity provider tokens created when depositing the funds on Uniswap, it cannot withdraw the liquidity which protects against a "rug pull". 

Here are some reasons creators and users might prefer using a launchpad over managing the process manually:

- **Simplicity** — No prior knowledge is required for token creation or DEX interaction.
- **Price Discovery** — The bonding curve offers a pricing mechanism even when the token has few or no holders.
- **Security** — Once deposited in the DEX liquidity cannot be withdrawn reducing the likelihood of a "rug pull" by the creator.
- **Guaranteed Liquidity** — Buyers on the bonding curve are assured liquidity if they decide to sell back into the curve.

## Differentiating Factors

Several features made our launchpad stand out from others:

- **Verified, Public, and Audited Protocol** — Our protocol's smart contract was fully verified, publicly accessible, and audited. In contrast most other launchpads used unverified code preventing users from reviewing it.
- **Engagement System for Token Visibility** — Users earned points with each trade which they could use to vote for a “Coin of the Day.” The selected token would gain prominent visibility across the platform for a day.
- **On-Chain Referral System** — Our launchpad offered a referral code system allowing users to include this code in transactions. Referrers received a portion of the fees which incentivized community growth.
- **Direct Uniswap Trading** — Once a token transitioned off the bonding curve users could seamlessly trade it on Uniswap directly through the launchpad.

## Design of the Launchpad

![Design of the Launchpad](/assets/token-launchpad.svg)

The main components of the system are shown above. Below we describe each component in more detail:

1. **Protocol Contracts**
    - Core smart contracts for the launchpad protocol which allow token creation, bonding curve trading, and promotion to Uniswap v2 pools.
2. **Frontend**
    - Supports key user actions such as:
        - Creating a new ERC20 token with a fixed supply on a bonding curve.
        - Browsing and trading tokens on the bonding curve or Uniswap.
        - Viewing holder distributions, user profiles by address, and token allocations.
        - Chatting with other users in real-time.
        - Voting for tokens using points to compete for “Coin of the Day”.
3. **CRUD API**
    - Powers frontend by accessing token, trade, and user data from the database.
    - Manages off-chain actions like voting, chatting, etc.
4. **Blockchain Listener**
    - Constantly observes the protocol contracts for new events (e.g., trades, token creations, promotions to Uniswap).
    - Updates relevant token states in the database and forwards these events to the websocket server for real-time frontend updates.
5. **Websocket Server**
    - Delivers real-time frontend updates for trades, token creations, voting, daily races, and chat interactions.
6. **Mentions Listener**
    - Monitors Twitter and Farcaster for posts mentioning a sentinel account which allow users to create tokens directly from those platforms.
    - When posts meet specified criteria (e.g., image, name, description, ticker), the listener triggers the creation of a token via the protocol contract.
7. **Cross-Launchpad Activity Listener**
    - Tracks tokens on other launchpads to stimulate activity during the launchpad's early stages.
    - Provides direct links to other launchpads for token trading.
8. **Uniswap Trade Component**
    - Abstraction layer for trading directly on Uniswap through the launchpad which used the Uniswap Universal Router.

## The Protocol Contracts

### The Bonding Curve

At the core of the contract was the bonding curve which determined token pricing at any given point. The bonding curve used was linear and was represented by the equation `y = m * x + b`, where:

- **x** — Total supply of tokens sold.
- **y** — Current price of the token being sold.
- **m** — Slope of the bonding curve.
- **b** — Starting price of the token sale.

From this formula, we derived key equations:

- **Token Price** — `y = m * x + b`
    - Calculates the current price of the token being sold.
- **Total Liquidity** — `total_liquidity = b * total_tradeable_supply + (m * total_tradeable_supply^2) / 2`
    - Represents the total ETH generated if all tokens are sold. This formula is based on the integral of the linear price equation and represents the area under the curve.
- **ETH Value for Trade** — `eth_value_for_trade = ((m * amount_traded^2) / 2) + m * x * amount_traded + b * amount_traded`
    - Calculates the amount of ETH required for a specific trade.
        - For a **buy** transaction `amount_traded` is positive.
        - For a **sell** transaction, `amount_traded` is negative.

### Promoting to Uniswap

Once all tokens had been sold and the bonding curve for a given token was fully funded with ETH the token would be promoted to Uniswap. For promotion there were two possible scenarios:

In the first scenario no Uniswap v2 pool existed for the token. Here the contract would handle the creation of the pool by funding it with the pre-set reserve tokens along with all the liquidity accumulated on the bonding curve. This setup ensured the initial price on Uniswap matched the final price of the bonding curve.

In the second scenario a Uniswap v2 pool for the token already existed and was created by an external actor. Even though the actor couldn’t add tokens due to transfer restrictions until promotion, they could deposit ETH into the pool. This could alter the ETH-to-token ratio which would influence the token's starting price. To prevent this the contract checked the ETH amount in the existing pool and minted additional reserve tokens as needed to maintain the expected ETH-to-token ratio.

Although the process was relatively straightforward implementing these safeguards was crucial to avoid potential exploits.

## The Uniswap Trade Component

This component was especially interesting because it allowed us to take fees in ETH in both directions while trading on a Uniswap v2 pool—something that’s normally not possible when interacting directly with v2 pool contracts. This was made possible by using the Uniswap Universal Router.

The Universal Router enables more complex trade interactions by using a set of encoded commands. While I give a brief overview of a trade example below, you can check the [GitHub repo here](https://github.com/Uniswap/universal-router) for more details on how it works.

We supported four main types of trades:

- Exact Input: ETH → ERC20
- Exact Input: ERC20 → ETH
- Exact Output: ETH → ERC20
- Exact Output: ERC20 → ETH

An “Exact Input” trade specifies the exact amount of input currency, with the output amount calculated based on that input. An “Exact Output” trade specifies the exact output amount, and the required input is calculated.

For example, an exact input ETH → ERC20 trade command flow using the Universal Router could work like this:

1. **Transfer Fee (optional)** — The `TRANSFER` command sends a portion of ETH as a fee to the fee wallet.
2. **Wrap ETH** — The `WRAP_ETH` command sends ETH to the router and turns it into wrapped ETH for the trade (since v2 pools only support wrapped ETH).
3. **V2 Swap** — The `V2_SWAP_EXACT_IN` command sets the exact amount of wrapped ETH to swap and specifies the minimum ERC20 output, accounting for slippage. It then transfers the output ERC20 to the user's wallet.
4. **Unwrap WETH** — If there is leftover ETH, it's unwrapped with the `UNWRAP_ETH` command.
5. **Transfer Remaining ETH** — Any leftover ETH is refunded to the user with the `TRANSFER` command.

## Final Thoughts

I hope this post gave you some insight into what a token launchpad is and how it is built. Thanks for reading! 
