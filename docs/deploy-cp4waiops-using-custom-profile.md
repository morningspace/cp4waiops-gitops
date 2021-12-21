<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Deploy CP4WAIOps Using Custom Profile](#deploy-cp4waiops-using-custom-profile)
  - [Step 1: Preparation](#step-1-preparation)
  - [Step 2: Deploy Prerequisites](#step-2-deploy-prerequisites)
  - [Step 3: Deploy Cloud Pak for Watson AIOps](#step-3-deploy-cloud-pak-for-watson-aiops)
  - [To Access Cloud Pak for Watson AIOps](#to-access-cloud-pak-for-watson-aiops)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Deploy CP4WAIOps Using Custom Profile

## Step 1: Preparation

Prepare one or two clusters to run Argo CD and Cloud Pak for Watson AIOps.

* If you use one cluster, then both Argo CD and Cloud Pak for Watson AIOps will be deployed on the same cluster
* If you use two clusters, then Argo CD and Cloud Pak for Watson AIOps can be run separately on each cluster

Let's say you use two clusters: 

* CLUSTER_A=`YOUR_ARGOCD_SERVER`
* CLUSTER_B=`YOUR_TARGET_SERVER`

Follow the steps below to prepare the environment:

* Install Argo CD on `CLUSTER_A` using Operator Hub by searching "OpenShift GitOps" then install.
* Login to `CLUSTER_A` from command line:

```shell
oc login --server=$CLUSTER_A -u kubeadmin
```

* Login to Argo CD from command line:

```shell
ARGO_PASSWORD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o "jsonpath={.data['admin\.password']}" | base64 -d)
ARGO_HOST=$(oc get route -n openshift-gitops openshift-gitops-server -o jsonpath={.spec.host})
argocd login --username admin --password $ARGO_PASSWORD $ARGO_HOST
```
* Login to `CLUSTER_B` from command line:

```
oc login --server=$CLUSTER_B -u kubeadmin
```

* Register `CLUSTER_B` as the target cluster to deploy Cloud Pak for Watson AIOps into Argo CD:

```
CURRENT_CONTEXT=$(oc config current-context)
argocd cluster add $CURRENT_CONTEXT
```

## Step 2: Deploy Prerequisites

Deploy storage from Argo CD UI by creating new Argo Application using below field values:

| Field            | Value
|:-----------------|:------
| Application Name | ceph
| Project          | default
| Repository URL   | https://github.com/morningspace/cp4waiops-gitops
| Revision         | dev
| Path             | ceph
| Cluster URL      | $CLUSTER_B
| Namespace        | rook-ceph

Deploy Resource Locker from Argo CD UI by creating new Argo Application using below field values:

| Field                     | Value
|:--------------------------|:------
| Application Name          | resource-locker
| Project                   | default
| Repository URL (Helm)     | https://redhat-cop.github.io/resource-locker-operator
| Chart                     | resource-locker-operator
| Version                   | v1.1.3
| Cluster URL               | $CLUSTER_B
| Namespace                 | resource-locker-operator

## Step 3: Deploy Cloud Pak for Watson AIOps

Deploy Cloud Pak for Watson AIOps from Argo CD UI by creating new Argo Application using below field values:

| Field                     | Value
|:--------------------------|:------
| Application Name          | cp4waiops
| Project                   | default
| Repository URL            | https://github.com/morningspace/cp4waiops-gitops
| Revision                  | dev
| Path                      | config/3.2/cp4waiops
| Cluster URL               | $CLUSTER_B
| Namespace                 | cp4waiops
| Helm:spec.dockerPassword  | Copy your password from https://myibm.ibm.com/products-services/containerlibrary

## To Access Cloud Pak for Watson AIOps

Run below commands to retrieve user, password, and the URL to access Cloud Pak for Watson AIOps:

```
oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_username}' | base64 -d && echo
oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 -d && echo
oc -n cp4waiops get route cpd -o jsonpath='{.spec.host}' && echo
```