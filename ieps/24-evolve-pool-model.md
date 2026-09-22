---
title: Evolve the Pool Model

iep-number: 24

creation-date: 2026-09-18

status: draft

authors:

- "@lukasfrank"

reviewers:

- "@main-reviewer-1"
- "@main-reviewer-2"

---

# IEP-24: Evolve the Pool Model

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [Current Model](#current-model)
    - [Target Model: Topology-Scoped Provisioning](#target-model-topology-scoped-provisioning)
    - [API Changes](#api-changes)
    - [Access Resolution via a Storage Plugin](#access-resolution-via-a-storage-plugin)
- [Alternatives](#alternatives)

## Summary

Remove the `VolumePool` and `BucketPool` API kinds and the storage scheduler
that assigns `Volume`s / `Bucket`s to them. Instead of scheduling a storage
resource onto an announced pool, a `Volume` / `Bucket` carries a **topology
constraint** (e.g. a zone) in its spec, and the `volumepoollet` /
`bucketpoollet` reconcile every resource whose topology matches the one they are
configured for. As part of the same change, a volume becomes an **opaque
handle**: the consumer resolves it by ID through a local storage plugin instead
of receiving backend access credentials, so the implementation is no longer
leaked to the tenant.

## Motivation

The pool model is a symmetric copy of compute's `MachinePool`: a provider
announces a `*Pool` describing its capabilities, and a scheduler matches pending
resources against it. For storage this indirection buys little:

* **Implementation independence.** Handing the tenant access credentials leaks
  the storage implementation to them. When a volume is instead an opaque handle
  with no exposed backend or credentials, its implementation can be replaced,
  migrated, or rebalanced without the user noticing.
* **A volume follows its machine.** A `Volume` is almost always attached to a
  `Machine` and must live in the same failure domain (zone). Compute already
  makes that placement decision; a second, independent scheduling step only
  risks putting the volume in a different zone than its machine.
* **Bucket pools are effectively global.** Object storage is a single region- or
  globally-scoped service; modelling it as a schedulable pool adds ceremony with
  no payoff.
* **API redundancy.** `VolumePool` / `BucketPool` are near-identical boilerplate
  (`ProviderID`, taints, state, conditions, available classes, capacity). What a
  consumer actually needs is expressible on the resource: class and topology.
* **Scheduler cost.** The storage scheduler and selector/toleration matching
  exist solely to serve this indirection.

### Goals

* Remove `VolumePool` / `BucketPool` as API kinds.
* Remove the storage scheduler and pool selector / toleration matching.
* Let a `Volume` / `Bucket` express placement directly via a topology constraint.
* Reconcile in the poollet by topology instead of by pool name.
* Stop shipping volume access credentials across the cluster: let the consumer
  resolve a volume by its ID through a local **storage plugin**.

### Non-Goals

* Changing the compute `MachinePool` / `machinebroker` model.

## Proposal

### Current Model

1. A `Volume` is created with `volumeClassRef` and either an explicit
   `volumePoolRef` or a `volumePoolSelector` (+ `tolerations`).
2. The storage scheduler (`internal/controllers/storage/scheduler`) matches the
   selector, tolerations, and each pool's `availableVolumeClasses`, and sets
   `spec.volumePoolRef`.
3. A `volumepoollet` started with `--volume-pool-name=<pool>` announces a
   `VolumePool` and watches every `Volume` whose `spec.volumePoolRef.name`
   equals its pool name, translating each into IRI calls to the broker.

`Bucket` / `BucketPool` / `bucketpoollet` mirror this exactly.

### Target Model: Topology-Scoped Provisioning

A `Volume` declares *what* it needs and *where* it should live; there is no pool
object and no scheduler:

```yaml
apiVersion: storage.ironcore.dev/v1alpha1
kind: Volume
metadata:
  name: db-data
spec:
  volumeClassRef:
    name: fast
  resources:
    storage: 10Gi
  topologyConstraints:          # constraint the provider must satisfy                
    matchLabels: # single zone
      topology.ironcore.dev/zone: a     
    matchExpressions:  # multi-zone
      - key: topology.ironcore.dev/zone
        operator: In
        values: [a, c] 
status:
  volumeID: <opaque-provider-handle>   # "where to find it"; resolved by the consumer plugin
  state: Available
```
[//]: # (@formatter:on)

The `volumepoollet` for zone `a` matches the topology, provisions the volume via
the `<ceph,*>-provider`, and writes back the opaque `status.volumeID` and
`state`. `volumeClassRef` + `resources` + `topology` are the complete placement
input.

### API Changes

* Remove the `VolumePool` and `BucketPool` kinds.
* On `Volume` and `Bucket`: drop pool selection (pool ref, pool selector,
  tolerations) and add a `topology` constraint expressing where the resource
  should live.
* A `Volume` no longer exposes access credentials in its status; a `Bucket` still
  exposes its endpoint and credentials, since the tenant's application consumes it
  directly.
* `VolumeClass` / `BucketClass` are unchanged.


### Access Resolution via a Storage Plugin

Today a volume's access credentials are handed to the consumer, so they leave the
storage domain and come to rest in a second provider. Because topology co-locates
the `Volume` with its `Machine`, the consumer already sits next to the storage
backend and can hold its own credentials for it. We therefore propose that the
consumer resolves the volume from its opaque handle using locally-held
credentials instead of being handed them, so credentials never leave the domain
that owns them.

This is what makes the volume an opaque handle, and it aligns
ironcore with the hyperscaler model: like an EBS volume, the tenant holds a
handle and never sees the backend or its credentials, while backup, clone and
migrate stay available through managed primitives (`VolumeSnapshot`,
`spec.dataSource`).


## Alternatives

* **Single `{key, value}` topology struct.** Simpler, but cannot pin region *and*
  zone; the map generalizes cleanly while staying readable for the common case.
* **Do nothing.** Retains a scheduler and two kinds whose only job is an
  indirection that is at best redundant and at worst races with compute.
