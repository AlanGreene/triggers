# Tekton Triggers Development Guide

## Getting started

1. [Ramp up on kubernetes and CRDs](#ramp-up-on-crds)
1. [Ramp Tekton Pipelines](#ramp-up-on-tekton-pipelines)
1. Create [a GitHub account](https://github.com/join)
1. Setup
   [GitHub access via SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)
1. [Create and checkout a repo fork](#checkout-your-fork)
1. Set up your [shell environment](#environment-setup)
1. Install [requirements](#requirements)
1. [Set up a Kubernetes cluster](#kubernetes-cluster)
1. [Configure kubectl to use your cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
1. [Set up a docker repository you can push to](https://github.com/knative/serving/blob/master/docs/setting-up-a-docker-registry.md)
1. [Install Tekton Pipelines](#install-pipelines)
1. [Install Tekton Triggers](#install-triggers)
1. [Iterate!](#iterating)

### Ramp up

Welcome to the project!! You may find these resources helpful to ramp up on some
of the technology this project is built on.

#### Ramp up on CRDs

This project extends Kubernetes (aka `k8s`) with Custom Resource Definitions
(CRDSs). To find out more:

- [The Kubernetes docs on Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) -
  These will orient you on what words like "Resource" and "Controller"
  concretely mean
- [Understanding Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/) -
  This will further solidify k8s nomenclature
- [API conventions - Types(kinds)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#types-kinds) -
  Another useful set of words describing words. "Objects" and "Lists" in k8s
  land
- [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)-
  A tutorial demonstrating how a Custom Resource Definition can be added to
  Kubernetes without anything actually "happening" beyond being able to list
  Objects of that kind

#### Ramp up on Tekton Pipelines

- [Tekton Pipelines README](https://github.com/tektoncd/pipeline/blob/master/docs/README.md) -
  Some of the terms here may make more sense!
- Install via
  [official installation docs](https://github.com/tektoncd/pipeline/blob/master/docs/install.md)
  or continue through [getting started for development](#getting-started)
- [Tekton Pipeline "Hello World" tutorial](https://tekton.dev/docs/getting-started/pipelines) -
  Define `Tasks`, `Pipelines`, and `PipelineResources`, see what happens when
  they are run

### Checkout your fork

The Go tools require that you clone the repository to the
`src/github.com/tektoncd/triggers` directory in your
[`GOPATH`](https://github.com/golang/go/wiki/SettingGOPATH).

To check out this repository:

1. Create your own
   [fork of this repo](https://help.github.com/articles/fork-a-repo/)
1. Clone it to your machine:

```shell
mkdir -p ${GOPATH}/src/github.com/tektoncd
cd ${GOPATH}/src/github.com/tektoncd
git clone git@github.com:${YOUR_GITHUB_USERNAME}/triggers.git
cd triggers
git remote add upstream git@github.com:tektoncd/triggers.git
git remote set-url --push upstream no_push
```

_Adding the `upstream` remote sets you up nicely for regularly
[syncing your fork](https://help.github.com/articles/syncing-a-fork/)._

### Requirements

You must install these tools:

1. [`go`](https://golang.org/doc/install): The language Tekton Pipelines is
   built in
1. [`git`](https://help.github.com/articles/set-up-git/): For source control
1. [`ko`](https://github.com/google/ko): For development. `ko` version v0.1 or
   higher is required for `triggers` to work correctly.
1. [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/): For
   interacting with your kube cluster

Your [`$GOPATH`] setting is critical for `ko apply` to function properly: a
successful run will typically involve building pushing images instead of only
configuring Kubernetes resources.

## Kubernetes cluster

Docker for Desktop using an edge version has been proven to work for both
developing and running Pipelines. The recommended configuration is:

- Kubernetes version 1.21 or later
- 4 vCPU nodes (`n1-standard-4`)
- Node autoscaling, up to 3 nodes
- API scopes for cloud-platform

To setup a cluster with GKE:

1. [Install required tools and setup GCP project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
   (You may find it useful to save the ID of the project in an environment
   variable (e.g. `PROJECT_ID`).

1. Create a GKE cluster (with `--cluster-version=latest` but you can use any
   version 1.21 or later):

   ```bash
   export PROJECT_ID=my-gcp-project
   export CLUSTER_NAME=mycoolcluster

   gcloud container clusters create $CLUSTER_NAME \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=3 \
    --scopes=cloud-platform \
    --enable-basic-auth \
    --no-issue-client-certificate \
    --project=$PROJECT_ID \
    --region=us-central1 \
    --machine-type=n1-standard-4 \
    --image-type=cos \
    --num-nodes=1 \
    --cluster-version=latest
   ```

   Note that
   [the `--scopes` argument to `gcloud container cluster create`](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create#--scopes)
   controls what GCP resources the cluster's default service account has access
   to; for example to give the default service account full access to your GCR
   registry, you can add `storage-full` to your `--scopes` arg.

1. Grant cluster-admin permissions to the current user:

   ```bash
   kubectl create clusterrolebinding cluster-admin-binding \
   --clusterrole=cluster-admin \
   --user=$(gcloud config get-value core/account)
   ```

## Environment Setup

To [run your controllers with `ko`](#install-triggers) you'll need to set these
environment variables (we recommend adding them to your `.bashrc`):

1. `GOPATH`: If you don't have one, simply pick a directory and add
   `export GOPATH=...`
1. `$GOPATH/bin` on `PATH`: This is so that tooling installed via `go get` will
   work properly.
1. `KO_DOCKER_REPO`: The docker repository to which developer images should be
   pushed (e.g. `gcr.io/[gcloud-project]`). You can also run a local registry
   and set `KO_DOCKER_REPO` to reference the registry (e.g. at
   `localhost:5000/myimages`).

`.bashrc` example:

```shell
export GOPATH="$HOME/go"
export PATH="${PATH}:${GOPATH}/bin"
export KO_DOCKER_REPO='gcr.io/my-gcloud-project-name'
```

Make sure to configure
[authentication](https://cloud.google.com/container-registry/docs/advanced-authentication#standalone_docker_credential_helper)
for your `KO_DOCKER_REPO` if required. To be able to push images to
`gcr.io/<project>`, you need to run this once:

```shell
gcloud auth configure-docker
```

The user you are using to interact with your k8s cluster must be a cluster admin
to create role bindings:

```shell
# Using gcloud to get your current user
USER=$(gcloud config get-value core/account)
# Make that user a cluster admin
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user="${USER}"
```

## Install Pipelines

To install [Tekton Pipelines](https://github.com/tektoncd/pipeline) you can
either:

- [Install a released version](https://github.com/tektoncd/pipeline/blob/master/docs/install.md)
- [Setup Tekton Pipelines for development](https://github.com/tektoncd/pipeline/blob/master/DEVELOPMENT.md)
  (install and iterate from HEAD)

## Iterating

While iterating on the project, you may need to:

1. [Install/Run Pipelines](#install-pipelines)
1. [Install/Run Triggers](#install-triggers)
1. Verify it's working by [looking at the logs](#accessing-logs)
1. Update your (external) dependencies with: `./hack/update-deps.sh`

   **Running dep ensure manually, will pull a bunch of scripts deleted
   [here](./hack/update-deps.sh#L29)**

1. Update your type definitions with: `./hack/update-codegen.sh`
1. Update the documentation with: `./hack/update-docs.sh`
1. [Add new CRD types](#adding-new-types)
1. [Add and run tests](./test/README.md#tests)

To make changes to these CRDs, you will probably interact with

- The CRD type definitions in
  [./pkg/apis/triggers/alpha1](./pkg/apis/triggers/v1alpha1)
- The reconcilers in [./pkg/reconciler](./pkg/reconciler)
- The clients are in [./pkg/client](./pkg/client) (these are generated by
  `./hack/update-codegen.sh`)

### Install Triggers

You can stand up a version of this controller on-cluster (to your
`kubectl config current-context`):

```shell
ko apply -f config/
ko apply -f config/interceptors
```

### Redeploy controller

As you make changes to the code, you can redeploy your controller with:

```shell
ko apply -f config/controller.yaml
```

### Tear it down

You can clean up everything with:

```shell
ko delete -f config/
```

## Accessing logs

To look at the controller logs, run:

```shell
kubectl -n tekton-pipelines logs $(kubectl -n tekton-pipelines get pods -l app=tekton-triggers-controller -o name)
```

To look at the webhook logs, run:

```shell
kubectl -n tekton-pipelines logs $(kubectl -n tekton-pipelines get pods -l app=tekton-triggers-webhook -o name)
```

## Adding new types

If you need to add a new CRD type, you will need to add:

1. A yaml definition in [config/](./config)
1. Add the type to the cluster roles in:
   - [200-clusterrole.yaml](./config/200-clusterrole.yaml)
   - [clusterrole-aggregate-edit.yaml](./config/clusterrole-aggregate-edit.yaml)
   - [clusterrole-aggregate-view.yaml](./config/clusterrole-aggregate-view.yaml)

## Adding feature gated API fields

We've introduced a feature-flag called `enable-api-fields` to the
[config-feature-flags.yaml file](config/config-feature-flags.yaml) deployed
as part of our releases.

This field can be configured either to be `alpha` or `stable`. This field is documented as 
part of our [install docs](install.md#customizing-the-triggers-controller-behavior).

For developers adding new features to Triggers' CRDs we've got
a couple of helpful tools to make gating those features simpler
and to provide a consistent testing experience.

### Guarding Features with Feature Gates

Writing new features is made trickier when you need to support both
the existing stable behaviour as well as your new alpha behaviour.

In the reconciler or sink code, you can guard your new features with an `if` statement
such as the following:

```go
alphaAPIEnabled := config.FromContextOrDefaults(ctx).FeatureFlags.EnableAPIFields == "alpha"
if alphaAPIEnabled {
  // new feature code goes here
} else {
  // existing stable code goes here
}
```

Notice that you'll need a context object to be passed into your function
for this to work. When writing new features keep in mind that you might
need to include this in your new function signatures.

### Guarding Validations with Feature Gates

Just because your application code might be correctly observing
the feature gate flag doesn't mean you're done yet! When a user submits
a Tekton resource it's validated by a validation webhook. Here too you'll need
to ensure your new features aren't accidentally accepted when the feature gate
suggests they shouldn't be. We've got a helper function,
[`ValidateEnabledAPIFields`](pkg/apis/triggers/v1beta1/version_validation.go),
to make validating the current feature gate easier. Use it like this:

```go
requiredVersion := config.AlphaAPIFields
// errs is an instance of *apis.FieldError, a common type in our validation code
errs = errs.Also(ValidateEnabledAPIFields(ctx, "your feature name", requiredVersion))
```

If the user's cluster isn't configured with the required feature gate it'll
return an error like this:

```
<your feature> requires "enable-api-fields" feature gate to be "alpha" but it is "stable"
```

### Unit Testing with Feature Gates

Any new code you write that uses the `ctx` context variable is trivially
unit tested with different feature gate settings. You should make sure
to unit test your code both with and without a feature gate enabled to
make sure it's properly guarded. See the following for an example of a
unit test that sets the feature gate to test behaviour:

```go
ctx, err := test.FeatureFlagsToContext(context.Background(), map[string]string{
        "enable-api-fields": "alpha",
})
if err != nil {
	t.Fatalf("unexpected error initializing feature flags: %v", err)
}

if err := ts.TestThing(ctx); err != nil {
	t.Errorf("unexpected error with alpha feature gate enabled: %v", err)
}
```

### Integration Tests

For integration tests we provide the [`requireGate` function](test/gate.go) which
should be passed to the `setup` function used by tests:

```go
c, namespace := setup(ctx, t, requireGate("enable-api-fields", "alpha"))
```

This will Skip your integration test if the feature gate is not set to `alpha`
with a clear message explaining why it was skipped.

**Note**: As with running example YAMLs you have to manually set the `enable-api-fields`
flag to `alpha` in your test cluster to see your alpha integration tests
run. When the flag in your cluster is `alpha` _all_ integration tests are executed,
both `stable` and `alpha`. Setting the feature flag to `stable` will exclude `alpha`
tests.
