---
layout: post
title: "Recovery Needs More Than a Retained Volume"
date: 2026-09-22
topic: "Recovery"
subtitle: "Kiln already preserved Lightning identity through Kubernetes lifecycle events. The next step was separating recovery data from the state it is supposed to recover."
---

Kiln has spent a lot of its recent work treating Lightning state differently from ordinary application state.

That has meant:

- a single LND replica
- `ReadWriteOncePod` storage fencing
- retained PVCs
- controlled StatefulSet updates
- graceful shutdown
- idempotent wallet initialization
- stable TLS identity
- destructive recovery testing

Those choices let an LND node survive pod replacement, operator restart, and even deletion and recreation of the `LightningNode` custom resource without becoming a different node.

That is an important recovery property.

But it is not a backup strategy.

A retained PVC and the Lightning state stored on it still share a failure domain. If the volume disappears, becomes corrupt, or the cluster itself is lost, "Kubernetes did not garbage-collect my PVC" is not much of a recovery plan.

The latest iteration starts separating those concerns.

## LND already has the right primitive

LND maintains a native Static Channel Backup, or SCB, in `channel.backup`.

The interesting part for Kiln is not inventing a new backup format. It is giving that existing recovery artifact lifecycle semantics that make sense in Kubernetes.

Kiln now publishes the SCB into a deterministic Secret:

```text
<lightning-node-name>-scb
```

For a node named `alice`, that becomes:

```text
alice-scb
```

The publisher watches LND's persisted `channel.backup` and continuously refreshes the Secret as the file changes.

That makes the recovery artifact visible outside the LND data volume.

## The backup must outlive the thing it backs up

The most important design choice is what Kiln does **not** do.

The SCB Secret does not have a `LightningNode` owner reference.

Normally, implementation resources created by an operator should be owned by their custom resource. Kiln follows that pattern for things like Services, StatefulSets, service accounts, and the restricted RPC credential Secrets it can recreate from persisted state.

A recovery artifact is different.

If the SCB Secret were owned by the `LightningNode`, deleting the custom resource could cause Kubernetes garbage collection to delete the backup at exactly the moment we are testing recovery.

So the relationship deliberately looks more like this:

```text
LightningNode/alice
       │
       ├── owns StatefulSet/alice
       ├── owns Service/alice
       ├── owns Secret/alice-rpc
       │
       └── publishes
              │
              ▼
       Secret/alice-scb
       retained independently
```

Kiln annotates the retained Secret so it knows which node the artifact belongs to, but it does not claim Kubernetes ownership over it.

That also means the controller has to be careful about adoption.

If a Secret named `alice-scb` already exists without Kiln's expected backup annotation, the operator refuses to silently take it over. A deterministic name should not become an excuse to overwrite unrelated data.

## Backup health is not node health

The new `BackupReady` condition is also intentionally separate from `Ready`.

A healthy LND node can be serving RPC correctly while the SCB publisher is temporarily behind or unable to refresh its destination Secret.

That is a real problem, but it is not the same problem as "the Lightning node is unavailable."

Kiln therefore reports both truths independently:

```text
Ready=True
BackupReady=False
```

can be a valid degraded recovery posture.

Once a non-empty `channel.backup` has been published:

```text
BackupReady=True
```

The condition tells an operator whether the recovery artifact exists without redefining runtime readiness around backup transport.

This is the same principle that has started showing up elsewhere in Kiln: conditions should describe distinct operational facts rather than collapsing everything into one green or red light.

## Green tests can still hide an upgrade bug

This iteration also produced a useful reminder about operator testing.

The first implementation went green.

Freshly created `LightningNode` resources got:

- the retained SCB Secret
- publisher RBAC for that Secret
- an updated publisher sidecar
- `BackupReady`
- destructive E2E coverage proving the SCB survived recovery

That looked complete.

It was not.

An existing node created by an older version of Kiln already had a StatefulSet and a publisher Role. The controller created the new SCB Secret, but it was not reconciling the existing Role or StatefulSet template toward the new publisher configuration.

A clean install passed. An upgrade could silently miss the feature.

That is exactly the kind of bug a Kubernetes operator should be suspicious of.

The fix was to reconcile the existing publisher configuration too:

- update the node-scoped Role so the publisher can write the SCB Secret
- update the StatefulSet pod template so the publisher knows about `channel.backup`
- preserve the existing `OnDelete` update strategy

The last point matters.

Kiln does not automatically restart an identity-bearing Lightning workload merely because the operator learned a new sidecar behavior. The desired StatefulSet template is updated, but the running pod keeps going until a deliberate pod replacement occurs.

After that replacement, the node comes back on the same retained state and starts publishing the SCB.

That is slower than an automatic rollout.

It is also much easier to reason about.

## Recovery tests should prove the artifact survives recovery

The destructive Lightning E2E now opens a real channel before checking the backup.

That distinction is important. An empty Secret created by the controller proves almost nothing.

The test now:

1. creates Alice and Bob
2. establishes the peer
3. funds a real channel
4. waits for the channel to become ready
5. waits for `BackupReady=True`
6. verifies `alice-scb` contains a non-empty `channel.backup`
7. records the SCB Secret UID
8. restarts the operator
9. verifies the same SCB remains
10. deletes and recreates the `LightningNode`
11. verifies the same retained PVC
12. verifies the same LND identity
13. verifies the same SCB Secret UID

That turns "we publish backups" into a lifecycle guarantee that can actually fail a build.

## The OpenShift demo tells the same story

The companion OpenShift demo is being updated around the same recovery contract.

It already demonstrated that Alice could:

- open a Kiln-managed channel to Bob
- make a real Lightning payment
- survive pod replacement
- survive complete `LightningNode` deletion and recreation
- return with the same identity, PVC, and channel

The SCB becomes another observable checkpoint in that walkthrough.

After the channel opens, the demo waits for:

```text
LightningNode/alice BackupReady=True
```

Then it inspects `Secret/alice-scb`, records its UID, replaces Alice's pod, deletes Alice's CR, recreates it, and proves the same retained backup artifact is still there.

That makes the recovery story much more concrete:

```text
persistent identity
      +
retained runtime state
      +
independent channel recovery artifact
```

Those are related guarantees, but they are not interchangeable.

## What this still does not solve

A Kubernetes Secret is not an independent disaster-recovery system.

The SCB has moved outside the Lightning PVC, but it can still live in the same cluster, the same control plane, and potentially the same underlying storage infrastructure.

This iteration does **not** yet provide:

- automatic off-cluster replication
- object-storage integration
- automated SCB restore
- seed custody
- recovery orchestration after total storage loss

Those are separate problems.

The useful milestone is that Kiln now has a first-class recovery artifact with explicit lifecycle semantics. Future external backup integration has something stable to consume instead of reaching into an LND filesystem and inventing its own ownership model.

The lesson from this pass is similar to the API work before it.

Kubernetes gives us good primitives, but their default lifecycle semantics are not automatically the right semantics for Bitcoin and Lightning infrastructure.

A PVC can be retained.

A Secret can be owned.

A StatefulSet can roll automatically.

For Kiln, each of those defaults has to be questioned against the thing we are actually trying to preserve.

Recovery begins when we stop confusing "the pod came back" with "the node can be recovered."
