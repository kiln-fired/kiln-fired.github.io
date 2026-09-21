---
layout: post
title: "Simulating the Bitcoin Network"
date: 2022-02-08
topic: "Simnet"
subtitle: "A private Bitcoin network made it possible to test Lightning behavior without putting real bitcoin at risk."
original_url: "https://www.openpersuasion.org/simulating-the-bitcoin-network/"
---

The next problem was testing. A Lightning node needs a functioning Bitcoin chain behind it, but experimenting with real funds is a poor development loop.

Kiln used `btcd` in `simnet` mode to create an isolated Bitcoin network. That solved the safety problem, but it exposed the next dependency: a fresh simulated chain starts at height zero. Lightning testing needs blocks for wallet funding, SegWit-era behavior, and channel open and close transactions.

That suggested a useful operator responsibility. Instead of manually entering a Bitcoin pod and generating blocks, the desired testing state could be expressed through the `BitcoinNode` resource. Kiln could then observe block height and drive the simulated chain toward a requested minimum.

The idea broadened from “deploy a node” to “manage enough node state to make the environment useful.” That distinction became important as Kiln evolved: reconciliation is valuable when it understands the protocol-specific state behind the workload.
