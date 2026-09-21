---
layout: post
title: "Drawing the API Boundary: Nodes, Peers, Channels, and References"
date: 2026-09-21
topic: "API design"
subtitle: "Kiln's latest iteration was less about adding resources and more about deciding which relationships deserve to exist in the Kubernetes API."
---

The latest Kiln work started with a simple question: are we still declaring infrastructure, or are we starting to turn LND commands into Kubernetes resources?

That question changed the direction of the API.

Invoices and payments are useful Lightning operations, but they are application intent. A platform team usually does not declare "someone should owe me 50,000 sats" as infrastructure desired state.

Peers and channels are different. They describe the topology and funded substrate of a Lightning node.

That led to a clearer boundary for Kiln:

- `BitcoinNode`
- `LightningNode`
- `LightningPeer`
- `LightningChannel`

The important part is not just the list of resources. It is how they reference each other.

## A reference graph, not an ownership tree

Kiln now models the infrastructure relationship as:

```text
BitcoinNode
    ↑ nodeRef
LightningNode
    ↑ nodeRef
LightningPeer
    ↑ peerRef
LightningChannel
```

Each resource references only its immediate first-class dependency.

A `LightningChannel` no longer repeats both a node and a peer reference. The peer already identifies its local `LightningNode`, so repeating the node created an unnecessary invariant that the controller had to police.

The API is simpler when:

```yaml
spec:
  peerRef: bob
  capacitySats: 100000
```

means exactly what it says.

The channel depends on the declared peer. The peer depends on a Lightning node. The Lightning node depends on its Bitcoin backend.

Those references are same-namespace, fixed-kind, and name-based. Kiln does not need a generic cross-namespace reference framework for this problem.

## References are not ownership

The resource graph deliberately does not use Kubernetes owner references between Kiln CRs.

Deleting a `BitcoinNode` should not cause Kubernetes garbage collection to tear through a `LightningNode`, a peer, and a funded channel.

Those resources have dependencies, but they are independently declared infrastructure.

Kiln uses owner references for implementation children it actually creates, such as StatefulSets, Services, and credential publishing resources. It does not use ownership to encode protocol relationships.

That distinction matters more as resources become financially meaningful.

## Managed Lightning derives its Bitcoin identity

The same cleanup exposed another duplicate.

A managed `LightningNode` previously declared both:

```yaml
bitcoinConnection:
  nodeRef: btcd
  network: simnet
```

But the referenced `BitcoinNode` already knows its network, RPC endpoint, TLS Secret, and authentication Secret.

The normalized form is now:

```yaml
bitcoinConnection:
  nodeRef: btcd
```

Kiln derives the rest.

An externally managed btcd node remains possible, but it is an explicit alternative under `bitcoinConnection.external`. The CRD requires exactly one of those two forms.

That keeps the managed-resource path genuinely declarative instead of asking users to repeat information the controller can resolve itself.

## Dependency changes should be events

Another lesson from the cleanup was that a dependency graph should behave like one.

Previously, several relationships relied mostly on periodic requeues. The controllers now index their reference fields and watch the resources they depend on.

Changes propagate:

```text
BitcoinNode
    ↓
LightningNode
    ↓
LightningPeer
    ↓
LightningChannel
```

That makes dependency state react immediately to Kubernetes events while still allowing periodic observation of external btcd and LND runtime state.

The distinction is useful: Kubernetes changes should be event-driven; protocol state still needs observation.

## Deletion is domain-specific

References do not automatically block deletion.

A missing `BitcoinNode` makes a dependent `LightningNode` unready. A missing `LightningNode` makes a peer wait. Those are dependency conditions, not ownership rules.

The `LightningPeer` to `LightningChannel` edge is different.

A funded channel depends on the peer identity. Kiln now explicitly blocks peer deletion while any `LightningChannel` still references it.

The lifecycle becomes:

```text
delete LightningPeer
        ↓
LightningChannel still exists
        ↓
DependencyBlocked
        ↓
channel closes and disappears
        ↓
peer disconnect resumes
        ↓
peer finalizer releases
```

That gives the API a meaningful safety rule instead of relying on an LND error as an accidental guardrail.

## A useful failure exposed a better readiness rule

The new dependency watches also surfaced something our older E2E test had been hiding.

During fast simnet mining, LND can briefly report `synced_to_chain=false` even when it is already at the latest block height. Before the new watches, Kiln could leave an older `syncedToChain=true` value in status long enough that the problem stayed invisible.

Once the dependency graph became event-driven, that transient state propagated immediately.

At first, the result was too aggressive:

- an already-connected peer became unready
- an already-funded channel stopped being observed

That was the wrong semantic.

The better rule is:

> Chain synchronization gates new side effects. It does not invalidate already-observed infrastructure.

So Kiln now requires chain sync before connecting a missing peer or funding a missing channel. But if LND still reports an existing peer or channel, Kiln keeps observing that state during transient sync changes.

That is a much better fit for reconciliation.

## Where the API boundary sits now

The current shape feels intentionally small:

```text
BitcoinNode
LightningNode
LightningPeer
LightningChannel
```

Invoices and payments stay outside the infrastructure CRD model. Applications can use the RPC surface Kiln publishes for those operations.

That gives the operator a clearer job:

- run Bitcoin infrastructure
- run Lightning infrastructure
- preserve identity-bearing state
- declare network topology
- declare funded channel relationships
- expose safe client access
- recover predictably

The most useful part of this iteration was not adding another feature.

It was deciding what Kiln should stop before becoming.
