---
layout: page
title: "On-Chain SVG Token Mint"
permalink: /projects/svg-mint
---

This project focused on creating a non-fungible token (NFT) contract where the image generation happened fully on-chain with minimal costs to users. The images were built as SVGs using basic tags like path, circle, ellipse, and group.

A set of traits (SVG components) and palettes (color schemes) were run-length encoded as bytes and loaded into the smart contract ahead of time. Users could construct their image by selecting traits and palettes on a web-app. This generated a seed, structured as follows:

```
    struct Seed {
        uint8 background;
        uint8 backgroundSubPalette;
        uint8 space;
        uint8 spaceSubPalette;
        bool skyEnabled;
        uint8 sky;
        uint8 skySubPalette;
        uint8 land;
        uint8 landSubPalette;
        bool structureEnabled;
        uint8 structure;
        uint8 structureSubPalette;
        bool characterEnabled;
        uint8 character;
        uint8 characterSubPalette;
    }

```

This seed was passed to the mint function, where the smart contract generated a customized NFT. Users only incurred the cost of storing the seed state. The SVG metadata was decoded and generated directly on-chain via a view function, making it free.

The project thus allowed users to customize and generate their NFT images on-chain at a low cost.
