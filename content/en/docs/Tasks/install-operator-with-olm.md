---
title: "Install your operator with OLM"
date: 2020-03-25
weight: 6
description: >
  [Once you've made your operator available in a catalog](/docs/tasks/make-operator-part-of-catalog/), or you've chosen an operator from the [list of operators available to be installed in the cluster](/docs/tasks/list-operators-available-to-install/), you can install your operator by creating a [`Subscription`](/docs/concepts/customresourcedefinitions/subscription/) to a specific [channel](/docs/concepts/glossary/#channel). 
---

## Prerequisites

Before installing an operator into a namespace, you will need to create an `OperatorGroup` that targets the namespaces your operator is planning to watch, to generate the required RBACs for your operator in those namespaces. You can read more about `OperatorGroup` [here](/docs/concepts/crds/operatorgroup). 

> Note: The namespaces targeted by the OperatorGroup must align with the `installModes` specified  in the `ClusterServiceVersion` of the operator's package. To know the `installModes` of an operator, inspect the packagemanifest: 
```bash
$ kubectl get packagemanifest <operator-name> -o jsonpath="{.status.channels[0].currentCSVDesc.installModes}"
          
```

> Note: This document uses a global OperatorGroup in the examples to install operators. To learn more about installing namespaced scoped operators, check out [operator scoping with OperatorGroups](/docs/advanced-tasks/operator-scoping-with-operatorgroups).  

## Install your operator

To install an Operator, simply create a `Subscription` for your operator. This represents the intent to subscribe to a stream of available versions of this Operator from a `CatalogSource`:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: <name-of-your-subscription>
  namespace: <namespace-you-want-your-operator-installed-in>
spec:
  channel: <channel-you-want-to-subscribe-to>
  name: <name-of-your-operator>
  source: <name-of-catalog-operator-is-part-of>
  sourceNamespace: <namespace-that-has-catalog>
  approval: <Automatic/Manual>
 ```  

You can read more about the `Subscription` object and what the different fields mean [here](/docs/concepts/crds/subscription).

The `Subscription` object creates an [InstallPlan](/docs/concepts/crds/installplan), which is either automatically approved (if `sub.spec.approval: Automatic`), or needs to be approved (if `sub.spec.approval: Manual`), following which the operator is installed in the namespace you want.

## Example: Install the latest version of an Operator

If you want to install an operator named `my-operator` in the namespace `foo` that is cluster scoped (i.e `installModes:AllNamespaces`), from a catalog named `my-catalog` that is in the namespace `olm`, and you want to subscribe to the channel `stable`, 

create a _global_ `OperatorGroup` (which selects all namespaces): 

```bash
$ cat og.yaml 

  apiVersion: operators.coreos.com/v1
  kind: OperatorGroup
  metadata:
    name: my-group
    namespace: foo

$ kubectl apply og.yaml 
  operatorgroup.operators.coreos.com/my-group created 
```

Then, create a subscription for the operator: 

```bash

$ cat sub.yaml

apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: sub-to-my-operator
  namespace: foo
spec:
  channel: stable
  name: my-operator
  source: my-catalog
  sourceNamespace: olm
  approval: Manual

$ kubectl apply -f sub.yaml
subscription.operators.coreos.com/sub-to-my-operator created
 ``` 

Since the `approval` is `Manual`, we need to manually go in and approve the `InstallPlan`

```bash 
$ kubectl get ip -n foo

NAME            CSV                   APPROVAL    APPROVED
install-nlwcw   my-operator.v0.9.2   Automatic     false

$ kubectl edit ip install-nlwcw -n foo
```
And then change the `spec.approved` from `false` to `true`

This should spin up the `ClusterServiceVersion` of the operator in the `foo` namespace`, following which the operator pod will spin up.

To ensure the operator installed successfully, check for the ClusterServiceVersion and the operator deployment in the namespace it was installed in. 

```
$ kubectl get csv -n <namespace-operator-was-installed-in>

NAME                  DISPLAY          VERSION           REPLACES              PHASE
<name-of-csv>     <operator-name>     <version>  <csv-of-previous-version>   Succeeded
```
```
$ kubectl get deployments -n <namespace-operator-was-installed-in>
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
<name-of-your-operator>      1/1     1            1           9m48s
```

If the ClusterServiceVersion fails to show up or does not reach the `Succeeded` phase, please check the [troubleshooting documentation](/docs/tasks/troubleshooting/clusterserviceversion/) to debug your installation.

## Example: Install a specific version of an Operator

If you want to install a particular version of your Operator, specify the `startingCSV` property in your `Subscription` like so:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: sub-to-my-operator
  namespace: foo
spec:
  channel: stable
  name: my-operator
  source: my-catalog
  sourceNamespace: olm
  approval: Manual
  startingCSV: 1.1.0
```

Notice that `approval` has been set to `Manual` as well in order to keep OLM from immediately updating your Operator, if `1.1.0` happens to be superseded by a newer version in `my-catalog`. Follow the instructions from the [previous example](#example-install-the-latest-version-of-an-operator) to approve the initial `InstallPlan` for this `Subscription`, so `1.1.0` is allowed to be installed.

If your intent is to pin an installed Operator to the particular version `1.1.0` you don't need to do anything. After approving the initial `InstallPlan` OLM will install version `1.1.0` of your Operator and keep it at that version. When updates are discovered in the catalog, OLM will wait not proceed unless you manual approve the update.
