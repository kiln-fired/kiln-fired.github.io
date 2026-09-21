---
layout: post
title: "Quick Update on Kiln: Bitcoin Simnet Experimentation"
date: 2022-08-03
topic: "Experiment"
subtitle: "CPU limits were not enough to make simnet mining behave like a realistic Bitcoin network."
original_url: "https://www.openpersuasion.org/quick-update-on-kiln-bitcoin-testnet-experimentation/"
---

The simulated network worked, but it worked too well.

With almost no mining competition, a local btcd simnet could produce blocks so quickly that Lightning nodes spent their time chasing chain synchronization instead of exercising channel operations. Kiln gained configurable Kubernetes CPU requests and limits in an attempt to slow mining down.

The result was revealing. Even extremely small CPU limits did not create a useful block cadence. The experiment showed that ordinary container resource controls were not a good substitute for the economics and difficulty dynamics of the real Bitcoin network.

The same round of work added basic Bitcoin peer configuration, making it easier to construct the multi-node topology simnet mining required. It also pointed toward `lndinit` as a promising building block for wallet initialization and key material.

That line of thinking survives in the modern Kiln work. Lifecycle and key custody are not incidental container concerns. They are part of the domain model, and the operator has to treat them deliberately.
