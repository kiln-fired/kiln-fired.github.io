---
layout: post
title: "Connecting a Kubernetes Operator to Bitcoin Node Management APIs"
date: 2022-02-10
topic: "RPC"
subtitle: "Kiln began observing Bitcoin node state directly through btcd's RPC API."
original_url: "https://www.openpersuasion.org/k8s-operator-bitcoin-node-api/"
---

Once Kiln needed to manage simulated chain state, it had to stop treating the Bitcoin node as a black box.

The `btcd` RPC API provided the bridge. Because both Kiln and btcd were written in Go, the operator could use native client libraries to query node state and invoke operations. The first useful observation was block height: read it from the node, then publish it into the `BitcoinNode` status.

That small step surfaced an operator-development lesson. Reconciliation happens while the world is changing. A custom resource can exist before its node is ready to accept RPC connections, so the controller must tolerate partial state and retry safely.

It also raised an architectural question that still matters: how tightly should an operator bind itself to a particular Bitcoin implementation? The original Kiln work chose btcd pragmatically, with interoperability left as a future design problem. The important progress was establishing direct, authenticated feedback between the controller and the protocol node it was responsible for.
