# Enforcing Service Mesh Structure using Gatekeeper

## Contents

- [Overview](#overview)
- [Project setup](#project-setup)
- [Setup Kubernetes and Istio](#setup-kubernetes-and-istio)
- [Install and configure Gatekeeper](#install-and-configure-gatekeeper)
- [Enforcing structural policies](#enforcing-structural-policies)
- [Cleanup](#cleanup)

## Overview

This repo contains a set of example policies that can be used to enforce specic service mesh structure. Specifically, the policies are managed by [OPA Gatekeeper](open-policy-agent/gatekeeper) and used to enforce specific production-friendly [Istio](http://istio.io) behaviors.

## Project setup

- Install the [Google Cloud SDK](https://cloud.google.com/sdk)
- Create a [Google Cloud](https://console.cloud.google.com) project (with billing)
- Enable the Kubernetes Engine [APIs](https://console.cloud.google.com/apis/library):

```
gcloud services enable container.googleapis.com
```

## Setup Kubernetes and Istio

- Create a GKE cluster

```
gcloud container clusters create [CLUSTER-NAME] \
  --cluster-version=latest \
  --machine-type=n1-standard-2
```

- Grab the cluster credentials so you can run `kubectl` commands

```
gcloud container clusters get-credentials [CLUSTER-NAME]
```

- Create a `cluster-admin` role binding so you can deploy Istio and Gatekeeper (later)

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```

- Download and unpack a recent version of Istio (e.g. `1.3.3`)

```
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.3 sh -
cd $ISTIO_VERSION
```
- Create the `istio-system` Namespace

```
kubectl create ns istio-system
```

- Use `helm` to install the Istio CRDs

```
helm template install/kubernetes/helm/istio-init \
  --name istio-init \
  --namespace istio-system | kubectl apply -f -
```

- Use `helm` to install the Istio control plane

```
helm template install/kubernetes/helm/istio \
  --name istio \
  --namespace istio-system \
  --set kiali.enabled=true \
  --set grafana.enabled=true \
  --set tracing.enabled=true | kubectl apply -f -
```

## Install and configure Gatekeeper

Refer to the [OPA Gatekeeper](open-policy-agent/gatekeeper) repo for docs and additional background on `Constraint` and `ConstraintTemplate` objects.

- Install the `gatekeeper` controller

```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

- Configure Gatekeeper to sync selected objects into it's cache
  - Required for `Constraints` that use `namespaceSelector` to match against
  - Required for multi-object policies that evaluate existing cluster- or namespace-scoped objects
  - Required for auditing existing resources

```
kubectl apply -f gatekeeper-config.yaml
```

## Enforcing structural policies

This repo contains 4 example policies in [`templates`](/templates) and [`constraints`](/constraints):

### Auditing services for not using correct port-naming convention

Checks `Service` objects in `Namespaces` labeled with `istio-injection: enabled`, and throws a violation if ports aren't named using [Istio conventions](https://istio.io/docs/setup/additional-setup/requirements/).

Upload the `ConstraintTemplate` and `Constraint`:

```
kubectl apply -f templates/port-name-template.yaml
kubectl apply -f constraints/port-name-constraint.yaml
```

Test the `Constraint` with the sample object:

```
kubectl apply -f sample-objects/bad-port-name.yaml
```

This `Constraint` set `enforcementAction: dryrun` so the object should be admitted to the cluster, and appear as an audit violation in the `status` field:

```
kubectl get allowedserviceportname.constraints.gatekeeper.sh port-name-constraint -o yaml
```

### Preventing VirtualService hostname matching collisions

Checks incoming `VirtualService` objects and compares them against existing `VirtualService` objects, and throws a violation if there are hostname/URI match collisions..

Upload the `ConstraintTemplate` and `Constraint`:

```
kubectl apply -f templates/vs-same-host-template.yaml
kubectl apply -f constraints/vs-same-host-constraint.yaml
```

Test the `Constraint` with the sample object:

```
kubectl apply -f sample-objects/bad-vs-host.yaml
```

This `Constraint` set `enforcementAction: dryrun` so the object should be admitted to the cluster, and appear as an audit violation in the `status` field:

```
kubectl get uniquevservicehostname.constraints.gatekeeper.sh unique-vs-host-constraint -o yaml
```

### Preventing mismatched mTLS authentication settings

Checks incoming `DestinationRule` objects and compares their mTLS settings against `Policy` object mTLS settings, and throws a violation if they don't match.

Upload the `ConstraintTemplate` and `Constraint`:

```
kubectl apply -f templates/mismatched-mtls-template.yaml
kubectl apply -f constraints/mismatched-mtls-constraint.yaml
```

Test the `Constraint` with the sample object:

```
kubectl apply -f sample-objects/mismatched-policy.yaml
kubectl apply -f sample-objects/mismatched-dr.yaml
```

This `Constraint` set `enforcementAction: dryrun` so the object should be admitted to the cluster, and appear as an audit violation in the `status` field:

```
kubectl get mismatchedmtls.constraints.gatekeeper.sh mismatched-mtls-constraint -o yaml
```

### Requiring services to disable unauthenticated access

Checks `ServiceRoleBinding` objects and throws a violation if they are set to allow unauthenticated access.

Upload the `ConstraintTemplate` and `Constraint`:

```
kubectl apply -f templates/source-all-template.yaml
kubectl apply -f constraints/source-all-constraint.yaml
```

Test the `Constraint` with the sample object:

```
kubectl apply -f sample-objects/bad-role-binding.yaml
```

This `Constraint` set `enforcementAction: deby` so the object should not be admitted to the cluster, and should return an error message.

## Cleanup

```
gcloud container clusters delete [CLUSTER-NAME]
```
