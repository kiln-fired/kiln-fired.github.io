---
layout: post
title: "Supporting Bitcoin and Lightning Protocols with Kubernetes"
date: 2022-02-07
topic: "Origin"
subtitle: "The original thesis behind Kiln: use the Kubernetes operator pattern to manage private Bitcoin and Lightning infrastructure."
original_url: "https://www.openpersuasion.org/bitcoin-and-lightning-on-kubernetes/"
---

Kiln started with a simple question: if organizations choose to participate directly in Bitcoin and Lightning, can Kubernetes take on some of the operational burden?

Direct participation means running infrastructure. A business that does not want to depend entirely on a third-party payment processor needs Bitcoin and Lightning nodes it can configure, observe, connect, and recover. Kubernetes already provides a declarative model for describing desired state, and the operator pattern lets that model learn a new domain.

The first Kiln experiments introduced custom resources for Bitcoin and Lightning nodes. A `BitcoinNode` described the RPC-facing Bitcoin backend. A `LightningNode` described an LND instance and its dependency on that backend. The operator reconciled those resources into running workloads and services.

The early implementation was intentionally small, but the shape was already visible: treat protocol infrastructure as something Kubernetes can understand, not merely something Kubernetes can launch.
