---
layout: post
title: "Incremental Steps Toward a Self-Contained Lightning Node Testing Environment"
date: 2022-02-19
topic: "Reconciliation"
subtitle: "Block production, retries, and idempotent reconciliation turned a loose simulation into a repeatable test environment."
original_url: "https://www.openpersuasion.org/toward-a-lightning-node-testing-environment/"
---

A useful Lightning test environment needs more than an isolated chain. It needs the chain to advance at the right moments.

Kiln added three pieces of simulated-network behavior: observe the current block height, generate blocks on demand until a requested minimum is reached, and optionally enable ongoing CPU mining. Those features were small, but they forced the controller to behave more like a real operator.

The key implementation pattern was requeueing. When one reconciliation step changes the world, it can be cleaner to stop and begin the loop again from the top. Delayed requeues also provide a natural way to retry node operations that are temporarily unavailable. That only works when every step is idempotent and termination conditions are explicit.

The mining experiment exposed another quirk: btcd simnet would not begin CPU mining without a peer. Once a second node joined, blocks arrived much too quickly. The testing problem had shifted from “produce blocks” to “produce them at a realistic pace.”
