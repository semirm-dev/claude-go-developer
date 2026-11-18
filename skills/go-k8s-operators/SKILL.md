---
name: go-k8s-operators
description: The user's rules for Kubernetes operators and controllers in Go with kubebuilder and controller-runtime - CRDs generated from Go structs, idempotent Reconcile, stale cache and 409 conflicts, requeueing, spec vs status subresource, save order, owner references vs finalizers, observedGeneration, and testing with fake client, envtest and kind. Use this whenever the user builds or reviews a Kubernetes controller, operator, CRD, reconcile loop, finalizer, or controller tests, or uses client-go/controller-runtime to watch resources.
---

# Kubernetes operators

An operator is a **CRD** (schema installed in the cluster), a **custom resource** (an instance with desired state) and a **controller** (Go program that makes the cluster match it). You don't always need a CRD: a controller can watch built-in kinds (Pods, Deployments, Nodes). Reference code is in `references/`.

## Scaffolding

Tools: `kubebuilder`, `controller-gen`, `kubectl`. Never hand-write the CRD; generate it from the Go struct.
```bash
kubebuilder init --domain example.com --repo github.com/org/project
kubebuilder create api --group apps --version v1 --kind Widget
# edit api/v1/widget_types.go
controller-gen object crd paths=./... output:crd:dir=config/crd/bases
kubectl apply -f config/crd/bases
go run ./cmd/main.go
kubectl apply -f config/samples/
```
Example: `references/g1-crd-and-controller/`

## Reconcile

- You get only namespace and name, **not the event** (create/update/delete). Read state, compare with spec, fix the difference.
- **Idempotent:** running it again with the same state changes nothing.
- `r.Get` reads from a local **cache** that may be stale. Updating a stale object returns **409 Conflict**: just return the error; the retry reads fresh state.
- Returning an error requeues with backoff. Use `ctrl.Result{RequeueAfter: 10 * time.Minute}` for periodic work.

Example: `references/g2-g3-reconcile/internal/widget/controller.go` (+ `conflict_test.go`)

## Spec and status

Spec is desired state (written by users); status is observed state (written by the controller).
- Add `// +kubebuilder:subresource:status` to the type, or `r.Status().Update()` doesn't work.
- **Save order matters:**
  1. `r.Update()` for metadata/spec (e.g. adding a finalizer). This refreshes your object from the API server, which still has the old status.
  2. `meta.SetStatusCondition()` changes status in memory only.
  3. `r.Status().Update()` persists the status.
  Any status set *before* step 1 is lost.
- Copy `metadata.generation` into `status.observedGeneration` when done, so readers know which spec the status describes.
- `meta.SetStatusCondition()` only bumps `lastTransitionTime` on a real change, so calling it every run is safe.

## Cleanup

- **In-cluster children** (ConfigMaps, Deployments): `controllerutil.SetControllerReference(owner, child, scheme)`; Kubernetes garbage-collects them.
- **External resources** (cloud APIs): add a **finalizer**. The object stays `Terminating` until the controller cleans up and removes the finalizer.
- Never remove a finalizer by hand to unstick an object; you leak whatever it protected.

## Testing

| Level | What | Use for |
|---|---|---|
| Fake client | In-memory store; call `Reconcile()` directly | Fast unit tests of reconcile logic |
| envtest | Real kube-apiserver + etcd locally, CRDs installed, real manager | Integration; clean up owned objects yourself (no GC) |
| kind | Real cluster | GC, finalizers, running pods end to end; slowest |

Example: `references/g4-testing/internal/widget/suite_test.go`

## Review checklist

- `Reconcile` that branches on what "happened" or isn't safe to run twice
- 409 conflicts handled with retries or re-gets instead of returning the error
- status written with `r.Update()`, status set before `r.Update()`, or no `+kubebuilder:subresource:status`
- `observedGeneration` never set
- in-cluster children without an owner reference; external resources without a finalizer
- a finalizer removed before cleanup succeeded
- a CRD YAML edited by hand instead of regenerated with `controller-gen`
