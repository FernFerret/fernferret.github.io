---
title: "Upgrading CRDs in Kubernetes"
date: 2022-02-05T15:22:47Z
draft: true
tags:
  - helm
  - crd
  - cert-manager
  - argocd
categories:
  - kubernetes
---

Today, I was performing a routine update in my Kubernetes cluster and I
encountered a unique error in ArgoCD. This was what I thought was a simple
update to [Cert Manager](https://cert-manager.io/docs/), from `1.6.1` to
`1.7.1`. Since most of the Helm charts I deploy use SemVer I didn't really
anticipate issues.
<!--more-->

## The Error

When I tried to sync, ArgoCD spat out a scary looking error message about all
the CRDs:

![Failed to sync CRDs](img/2022-02-05-16-19-12.png)

I'd missed the note on the [ArtifactHub
page](https://artifacthub.io/packages/helm/cert-manager/cert-manager#upgrading-the-chart)
stating that I should check the release notes for [Cert
Manager](https://github.com/cert-manager/cert-manager/releases/tag/v1.7.0).
Turns out, they'd removed a bunch of old CRDs in this release. That's cool, they
also mention using their tool `cmctl` to easily do this.

However I thought, what if I needed to do this with one of my own CRDs or
someone else had a CRD that we needed to perform this change on. Lucky for us
the Kubernetes documentation has a section on [upgrading existing object to a new stored version](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#upgrade-existing-objects-to-a-new-stored-version).

This document states:

> The API server records each version which has ever been marked as the storage
> version in the status field storedVersions. Objects may have been persisted at
> any version that has ever been designated as a storage version. No objects can
> exist in storage at a version that has never been a storage version.

## Deep Dive into `storedVersions`

Let's have a look at `status.storedVersions` in my `ClusterIssuers` CRD, since I
know I have one.

```text
kubectl get crd clusterissuers.cert-manager.io -o yaml | \
  yq e '.status.storedVersions' -
- v1alpha2
- v1
```

This means that at some point I've had a `v1alpha2` and a `v1` version of this
CRD installed. I'm sure the `cmctl` binary described earlier would do the
following for me, but just to follow things all the way through, let's use the
stock Kubernetes commands to migrate these CRDs.

As a quick note, another way to do this that's a lot more harmful is to just
delete the CRD, but this will also delete any instances of it, which is **very
bad** if you like having your cluster work.

The process is effectively:

* verify that we don't have any more of the old APIs
* patch the status to remove the clusters' memory of them
* upgrade to a version of the CRD that removes old versions

## Fixing our CRD

The reason this is "broken" is that the new version no longer has `v1alpha2` in
it, so Kubernetes is trying to protect us by saying "hey man, at some point you
used `v1alpha2` in your cluster, but you're trying to remove it from the CRD!"
Thanks [Captain Kube](https://www.cncf.io/phippy/)!

We can use the `patch` operation combined with `kube proxy` [as stated in the
docs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#upgrade-existing-objects-to-a-new-stored-version)
to remove the `v1alpha2` CRD `storedVersion`:

```text
kubectl proxy &
curl --header "Content-Type: application/json-patch+json" \
  --request PATCH http://localhost:8001/apis/apiextensions.k8s.io/v1/customresourcedefinitions//status \
  --data '[{"op": "replace", "path": "/status/storedVersions", "value":["v1"]}]'
```

This will return the CRD and you should be able to verify it no longer has the
old `storedVersion`:

```text
kubectl get crd clusterissuers.cert-manager.io -o yaml | \
  yq e '.status.storedVersions' -
- v1
```

Awesome, we're ready to rock now and can go sync ArgoCD!