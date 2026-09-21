---
layout: post
title: "Kiln Fired Again: From Runtime Modernization to a Real Lifecycle Contract"
date: 2026-09-20
topic: "Modernization"
subtitle: "A day of tightening Kiln from a collection of Kubernetes workloads into a more explicit operating contract for btcd and LND."
---

Today was less about adding flashy features and more about making Kiln behave like infrastructure we could actually trust.

The project already knew how to create Bitcoin and Lightning workloads. The latest iterations focused on what happens after that first successful reconcile: pods move, controllers restart, credentials get republished, custom resources are deleted, storage survives, and eventually someone points the same APIs at a network where mistakes matter.

That work changed Kiln in several important ways.

## Moving the Bitcoin runtime upstream

One of the first cleanup items was removing an unnecessary dependency on a project-maintained btcd image.

Kiln now defaults directly to the upstream btcd image:

`ghcr.io/btcsuite/btcd:v0.26.2`

That sounds small, but it removes a supply-chain responsibility from the project. Kiln should own the operator behavior, not a redundant repackaging of btcd.

The runtime configuration was tightened at the same time so the image, command-line flags, probes, and generated manifests all agree on how btcd is started.

This also forced a useful design decision: Kiln is not pretending to be implementation-agnostic right now. The current Bitcoin implementation is btcd and the current Lightning implementation is LND. We can generalize later if a real requirement earns that abstraction.

## Treating Bitcoin as stateful infrastructure

The next iteration hardened `BitcoinNode`.

A Bitcoin node can reconstruct its chain state, but that does not mean Kubernetes should casually destroy or race its storage. The controller now uses stronger StatefulSet semantics, explicit lifecycle conditions, and finalization behavior.

Deletion is no longer just "the custom resource disappeared." Kiln coordinates workload teardown and exposes lifecycle state through status.

That gives the controller a much clearer contract around what it owns and what Kubernetes is allowed to replace.

## Giving Lightning a stronger recovery contract

Lightning needed more care.

An LND node has identity-bearing state. Wallet state, channel state, macaroons, and TLS identity are not equivalent to a cache that can be regenerated from the network.

The `LightningNode` controller now models that distinction directly.

Kiln uses a single-replica StatefulSet, `ReadWriteOncePod` storage fencing, retained PVCs, graceful shutdown, and an `OnDelete` update strategy. Wallet initialization through `lndinit` is idempotent, so an existing wallet on the retained volume is reused rather than silently replaced.

A `LightningNode` can also reference a `BitcoinNode` directly. Kiln waits for that dependency to become usable before starting LND and derives the in-cluster btcd service and credential references from the resource relationship.

The important part is the failure path. Deleting and recreating the same `LightningNode` can reattach the retained storage and recover the same LND identity.

That is now part of the contract, not an accident.

## Publishing restricted Lightning client credentials

Once LND could recover predictably, the next question was how clients should connect to it.

Kiln now publishes a dedicated RPC Secret containing:

- the LND TLS certificate
- a read-only macaroon
- an invoice macaroon

The admin macaroon is intentionally not exported.

The publisher also runs with narrowly scoped RBAC and refuses to adopt unrelated Secrets. That keeps credential publication from becoming a generic "controller can mutate any Secret in the namespace" capability.

This work also exposed a TLS detail that mattered in Kubernetes. LND's certificate needs to remain valid for the stable Service DNS name, not just whatever pod address happened to exist when the node started.

Kiln now configures LND accordingly and keeps the TLS identity stable across pod replacement.

## Proving recovery in a real cluster

Unit and envtest coverage are useful, but they cannot prove that a real btcd + LND stack survives destructive Kubernetes operations.

We added a kind-based destructive E2E test that creates the full stack and checks recovery across:

1. normal startup
2. operator restart
3. LND pod deletion and recreation
4. RPC credential Secret deletion and republishing
5. complete `LightningNode` deletion
6. `LightningNode` recreation against the retained PVC

The test also verifies that an independent client can authenticate to LND through the published Service and credentials before and after recovery.

The first successful run took a little over ten minutes. That was valuable feedback too. A destructive integration test that expensive should not run for a documentation-only pull request.

The workflow is now scoped to runtime and operator changes, runs nightly on `main`, and remains manually dispatchable.

That feels like the right boundary: keep the test serious without making every small change pay the full cluster-recovery tax.

## Using LND itself as the source of truth

A ready Kubernetes pod is not proof that LND is healthy.

Kiln now performs an authenticated `GetInfo` call against LND and treats that as the authoritative runtime readiness signal.

The `LightningNode` status can expose the node identity, alias, version, block height, chain synchronization, graph synchronization, peer count, and channel counts.

If Kubernetes says the pod is ready but authenticated LND RPC fails, Kiln reports the node as degraded.

That closes an important gap between container health and protocol health.

## Adding explicit network and mainnet guardrails

The latest iteration was about making network selection an explicit safety boundary.

Both Bitcoin and Lightning resources now default to `simnet`. Supported networks are `simnet`, `testnet`, `regtest`, `signet`, and `mainnet`.

Mainnet requires an explicit opt-in:

```yaml
spec:
  network: mainnet
  safety:
    allowMainnet: true
```

For Lightning, the selected network must also match the referenced `BitcoinNode`. Kiln refuses to start LND if the two resources resolve to different networks.

The controller exposes a `NetworkReady` condition and reports the resolved network in status.

Development conveniences such as automatic block generation and CPU mining are also blocked on mainnet.

If a previously running resource becomes invalid under the network policy, Kiln removes the workload gracefully while retaining the PVC. Restoring a valid policy allows the persisted node to recover.

The network field itself is immutable so a node cannot quietly transition from one chain context to another underneath the same persisted identity.

## Where that leaves Kiln

The project has a much more coherent shape now.

Kiln is not trying to be a universal Bitcoin abstraction layer. It is a focused Kubernetes operator for btcd and LND with explicit behavior around persistence, dependencies, credentials, protocol-level readiness, recovery, and network safety.

The current APIs are still small:

- `BitcoinNode`
- `LightningNode`
- `Seed`

But the semantics behind them are becoming much less casual.

That is the direction I want for the project. Fewer implicit assumptions, fewer disposable-state shortcuts, and more behavior that can be stated, tested, and recovered from.
