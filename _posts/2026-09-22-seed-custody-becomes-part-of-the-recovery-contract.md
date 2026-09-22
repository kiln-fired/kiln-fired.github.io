---
layout: post
title: "Seed Custody Becomes Part of the Recovery Contract"
date: 2026-09-22
topic: "Recovery"
subtitle: "Kiln is moving seed material out of custom resources and giving it lifecycle semantics that match what it actually represents: recovery-critical identity."
---

The recent recovery work in Kiln has mostly focused on the state around a running Lightning node.

Retained PVCs preserve the LND wallet and channel database. Static Channel Backups are published outside that volume so the recovery artifact does not disappear with the state it is supposed to recover.

The next layer down is even more fundamental: the wallet seed itself.

Kiln already had a first-class `Seed` resource, but its lifecycle semantics were still too casual for recovery-critical material.

The latest iteration changes that.

## The old API put secrets in the wrong place

The original `Seed` API allowed the mnemonic and passphrase directly in the custom resource:

```yaml
spec:
  secretName: alice-seed
  mnemonic: "..."
  passphrase: "..."
```

That is convenient for a prototype, but it is not a good security boundary.

A custom resource is ordinary Kubernetes API data. Anyone with permission to read `Seed` objects can read those values, and the values are persisted as part of the resource.

Kubernetes Secrets are not a complete custody system either, but they at least fit the cluster's normal RBAC, encryption-at-rest, admission, audit, and external-secret integration patterns.

So the Seed API no longer carries the sensitive values itself.

Existing seed material is now imported by reference:

```yaml
apiVersion: bitcoin.kiln-fired.github.io/v1alpha1
kind: Seed
metadata:
  name: alice
spec:
  secretName: alice-seed
  network: simnet
  import:
    secretName: alice-seed-import
    mnemonicKey: mnemonic
    passphraseKey: passphrase
```

If `import` is omitted, Kiln still generates new seed material.

The important difference is where the sensitive bytes live.

## The output Secret should outlive the Seed CR

The older controller created the output Secret as a normal child of the `Seed` resource.

That meant Kubernetes ownership worked exactly as designed:

```text
Seed/alice
    owns
      |
      v
Secret/alice-seed
```

The problem is that this is the wrong lifecycle for recovery material.

Deleting the `Seed` CR could garbage-collect the thing needed to recreate the wallet identity later.

The new relationship is intentionally different:

```text
Seed/alice
    publishes
      |
      v
Secret/alice-seed
retained independently
```

The output Secret has no controller owner reference.

Kiln labels and annotates it as retained seed material, so a recreated `Seed` resource can recognize and reuse it without treating every same-named Secret as safe to adopt.

That mirrors the design we just introduced for LND Static Channel Backups.

Recovery data should not automatically share the lifecycle of the resource that produced it.

## Retention also needs an adoption rule

Removing an owner reference is easy.

Doing it safely is more interesting.

If `Secret/alice-seed` already exists, Kiln cannot assume it belongs to `Seed/alice` just because the names line up.

The controller now distinguishes three cases:

1. A Secret explicitly marked as retained material for this Seed can be reused.
2. A legacy Secret still controller-owned by the current Seed can be migrated in place.
3. An unrelated same-name Secret is a collision and is not adopted.

The migration path matters because existing Kiln users should not need to rotate seed material just to receive safer lifecycle behavior.

When Kiln sees an older controller-owned Seed Secret, it removes the old controller owner reference, adds the retained-seed marker, and preserves the bytes exactly.

That turns the stronger recovery contract into an upgrade instead of a reset.

## Losing a generated seed must not create a new identity

There is another failure mode that is easy to miss.

Suppose Kiln generates a seed, reports the resource ready, and publishes `Secret/alice-seed`.

Later, somebody deletes that Secret.

The naive reconciliation loop would observe "desired Secret missing" and generate another one.

That would make the Kubernetes resource look healthy again.

It would also silently create a different wallet identity under the same `Seed` name.

Kiln now refuses to do that.

Once a generated Seed has reached `Ready=True`, a missing retained output Secret becomes:

```text
Ready=False
Reason=SeedMaterialLost
```

The controller does not invent replacement recovery material.

Imported seeds are different. If the output Secret disappears but the source import Secret still exists, Kiln can reproduce the same seed material deterministically and republish the output.

That distinction is now explicit:

```text
generated seed lost
    -> cannot reconstruct safely

imported seed output lost
    -> reconstruct from declared source Secret
```

This is the kind of distinction that matters when reconciliation is managing identity instead of disposable configuration.

## Seed status now describes operational reality

The old `SeedStatus` was empty.

Errors such as an invalid mnemonic or incorrect passphrase were reconciliation failures, but the resource itself did not explain what was wrong.

Kiln now reports conditions including:

- `Ready`
- `SecretReady`

and non-sensitive reasons such as:

- `InvalidMnemonic`
- `InvalidPassphrase`
- `ImportSecretNotFound`
- `ImportSecretInvalid`
- `SecretCollision`
- `SeedMaterialLost`

The status deliberately does not echo mnemonic or passphrase values.

That gives operators useful state without turning status into another secret transport.

## The demo now proves the lifecycle too

The OpenShift demo has a useful constraint: Alice intentionally uses a known public simnet seed because the demo reward address is tied to that wallet.

Previously, that fixture lived directly in the `Seed` CR.

The updated flow materializes the fixture into a Kubernetes Secret first:

```text
public simnet fixture
        |
        v
Secret/alice-seed-import
        |
        | spec.import
        v
Seed/alice
        |
        | publishes
        v
Secret/alice-seed
retained independently
```

The walkthrough then proves the retention behavior instead of merely assuming it:

1. wait for `Seed/alice Ready=True`
2. record the UID of `Secret/alice-seed`
3. verify it has no controller owner reference
4. delete `Seed/alice`
5. verify the output Secret still exists
6. recreate `Seed/alice`
7. verify the same Secret UID remains

That gives the demo another recovery checkpoint alongside the retained Lightning PVC and retained SCB.

## What this still does not solve

A Kubernetes Secret is not a hardware wallet, HSM, Vault deployment, or independent disaster-recovery system.

This iteration does not yet provide:

- external secret-manager integration
- hardware-backed key custody
- automatic off-cluster seed replication
- multi-party recovery
- automatic wallet restoration after total cluster loss

Those are separate capabilities.

The useful change is that Kiln now has a much cleaner boundary for them.

The `Seed` resource declares how seed material should be produced or imported. Sensitive input lives behind a Secret reference. The output is treated as retained recovery material. Status reports whether that contract is intact.

That is a better foundation for stronger custody later.

## Recovery is becoming a system property

The interesting pattern across the last few Kiln iterations is that recovery is no longer one feature.

It is showing up in multiple parts of the control plane:

```text
Lightning identity
    -> retained PVC

channel recovery
    -> retained SCB Secret

wallet identity recovery
    -> retained Seed Secret
```

Each artifact has a different role, but the lifecycle principle is the same.

Kubernetes ownership and garbage collection are powerful defaults. They are not automatically the right defaults for Bitcoin and Lightning recovery material.

The operator has to know which state is replaceable, which state is reconstructable, and which state must never be silently regenerated.

That distinction is becoming part of Kiln's API contract rather than tribal knowledge around how to operate it.
