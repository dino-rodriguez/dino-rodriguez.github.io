---
layout: page
title: "Constant Rate Auction Token Mint"
permalink: /projects/cra-mint
---

For this project I implemented a smart contract for token sales using a modified version of the Constant Rate Issuance Sales Protocol (CRISP) developed by Paradigm. You can read more about CRISP [here](https://www.paradigm.xyz/2022/01/constant-rate-issuance-sales-protocol).

The idea behind CRISP is to maintain a constant token sale rate by adjusting prices based on demand. By tracking the time elapsed between sales, the contract adjusts prices upward if demand is high, and downward if demand is low.

In practice, the variable pricing kept users engaged, resulting in thousands of interactions and a complete sell-out of all tokens.
