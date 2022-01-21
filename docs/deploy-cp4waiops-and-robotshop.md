<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Deploy Extremely Small CP4WAIOps with Sample App for Trial Use](#deploy-extremely-small-cp4waiops-with-sample-app-for-trial-use)
  - [Prepare Environment](#prepare-environment)
  - [Deploy Argo CD](#deploy-argo-cd)
  - [Deploy Ceph Storage](#deploy-ceph-storage)
  - [Deploy CP4WAIOps](#deploy-cp4waiops)
  - [Deploy Robot Shop](#deploy-robot-shop)
  - [Deploy Humio and Fluent Bit](#deploy-humio-and-fluent-bit)
  - [Deploy Istio](#deploy-istio)
  - [Access Applications](#access-applications)
  - [Demo: Log Anomaly Detection](#demo-log-anomaly-detection)
    - [Define dynamic template to create resource groups for each service in Robot Shop](#define-dynamic-template-to-create-resource-groups-for-each-service-in-robot-shop)
    - [Create application for Robot Shop](#create-application-for-robot-shop)
    - [Enable Humio live log and trigger applicaton story/alert](#enable-humio-live-log-and-trigger-applicaton-storyalert)
    - [Inject 401 issue into ratings service in Robot Shop](#inject-401-issue-into-ratings-service-in-robot-shop)
    - [Verify log anomaly detecting in application](#verify-log-anomaly-detecting-in-application)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Deploy Extremely Small CP4WAIOps with Sample App for Trial Use

>❗️Note: This is mainly used for sandbox with restricted resource to deploy CP4WAIOps for PoC, demo, or dev environment. It is not for production use. The steps are verified using CP4WAIOps v3.2. The document is still WIP. Feel free to submit issue or PR.

This tutorial is aimed to work you through the steps to deploy an extremely small Cloud Pak for Watson AIOps (CP4WAIOps) with Robot Shop as sample application for trial use. It includes:

- How to deploy CP4WAIOps with custom sized profile using Argo CD.
- How to deploy Robot Shop as sample application and its dependencies to demonstrate the use of CP4WAIOps.
- How to configure CP4WAIOps to go through its major features (Golden Thread).

## Prepare Environment

Prepare an OCP cluster as an all-in-one sandbox to run all the necessary softwares, including:

- Argo CD
- Ceph storage
- CP4WAIOps
- Robot Shop
- Humio
- Fluent Bit
- Istio (for fault injection)

The cluster needs 3 worker nodes where each node has 16 core CPU and 32GB memory.

## Deploy Argo CD

To deploy Argo CD, log on cluster via OCP console, navigate to `Operators` -> `OperatorHub`, search for `OpenShift GitOps`, then install using default settings.

After it's finished, navigate to Argo CD UI by clicking `Cluster Argo CD` from the drop down menu on top right of OCP Console, choose `Log in via OpenShift`, then choose `Allow selected permissions`.

Create an Argo Application named `argocd-custom` using below field values to customize Argo behaviors such as custom health checks:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | argocd-custom                                         |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Repository URL        | https://github.com/morningspace/cp4waiops-gitops      |
| Revision              | dev                                                   |
| Path                  | config/3.2/argocd/openshift                           |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | openshift-gitops                                      |

Create an Argo Application named `resource-locker` using below field values to allow Argo CD to override default cpu and memory settings for CP4WAIOps:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | resource-locker                                       |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Sync Option           | CreateNamespace=true                                  |
| Repository URL (Helm) | https://redhat-cop.github.io/resource-locker-operator |
| Chart                 | resource-locker-operator                              |
| Version               | v1.1.3                                                |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | resource-locker-operator                              |

## Deploy Ceph Storage

Create an Argo Application named `ceph` using below field values as default storage for other applications run in this cluster including CP4WAIOps, Robot Shop, Humio, etc.:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | ceph                                                  |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Repository URL        | https://github.com/morningspace/cp4waiops-gitops      |
| Revision              | dev                                                   |
| Path                  | ceph                                                  |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | rook-ceph                                             |

Make sure the Application is `Healthy` and `Synced` before start to deploy CP4WAIOps. (Click `SYNC` to enforce sync when you see it is `OutOfSync` but has already been `Healthy`)

## Deploy CP4WAIOps

Create an Argo Application named `cp4waiops` using below field values to start the deployment of CP4WAIOps:

| Field                     | Value                                             |
| ------------------------- |-------------------------------------------------- |
| Application Name          | cp4waiops                                         |
| Project                   | default                                           |
| Sync Policy               | Automatic                                         |
| Repository URL            | https://github.com/morningspace/cp4waiops-gitops  |
| Revision                  | dev                                               |
| Path                      | config/3.2/cp4waiops                              |
| Cluster URL               | https://kubernetes.default.svc                    |
| Namespace                 | cp4waiops                                         |
| Helm: spec.dockerPassword | Replace by your own password copied from [here]   |

[here]: https://myibm.ibm.com/products-services/containerlibrary

This will take approximately less than 1 hour to finish. When you see the Application is `Healthy` and `Synced`, it means CP4WAIOps install is completed.

## Deploy Robot Shop

Create Argo Application `robot-shop-prereq` and `robot-shop` using below field values to deploy Robot Shop as a sample application for CP4WAIOps to demonstrate its features:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | robot-shop-prereq                                     |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Sync Option           | CreateNamespace=true                                  |
| Repository URL        | https://github.com/morningspace/sample-app-gitops     |
| Revision              | HEAD                                                  |
| Path                  | config/apps/robot-shop                                |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | robot-shop                                            |

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | robot-shop                                            |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Repository URL        | https://github.com/instana/robot-shop                 |
| Revision              | HEAD                                                  |
| Path                  | K8s/helm                                              |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | robot-shop                                            |
| Helm: nodeport        | true                                                  |
| Helm: ocCreateRoute   | true                                                  |
| Helm: openshift       | true                                                  |

This will take approximately less than 10 minues to finish. When you see the Applications are `Healthy` and `Synced`, it means Robot Shop install is completed.

## Deploy Humio and Fluent Bit

Create an Argo Application named `humio` using below field values to start the deployment of Humio and Fluent Bit:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Application Name      | humio                                                 |
| Project               | default                                               |
| Sync Policy           | Automatic                                             |
| Sync Option           | CreateNamespace=true                                  |
| Repository URL        | https://github.com/cloud-pak-gitops/sample-app-gitops |
| Revision              | HEAD                                                  |
| Path                  | gitops-charts/humio-helm-charts                       |
| Cluster URL           | https://kubernetes.default.svc                        |
| Namespace             | humio-logging                                         |
| Helm: humio-core.openshift.host | The cluster domain name                     |

Use below command to get the cluster domain name:

```sh
kubectl get ingresscontrollers default -n openshift-ingress-operator -o jsonpath="{.status.domain}" | cut -c6-
```

This will take approximately less than 15 minues to finish. When you see the Application is `Healthy` and `Synced`, it means Humio and Fluent Bit install are completed.

## Deploy Istio

WIP

## Access Applications

To access CP4WAIOps, run below command to get the access information:

```sh
# Access URL
kubectl -n cp4waiops get installation aiops-installation -o jsonpath='{.status.locations.cloudPakUiUrl}{"\n"}'
# Password for user admin
oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 -d
```

To access Robot Shop, run below command to get the access information:

```sh
# Access URL
kubectl -n robot-shop get route web -o jsonpath='{"http://"}{.spec.host}{"\n"}'
```

To access Humio, run below commands to get the access information:

```sh
# Access URL
kubectl -n humio-logging get route humio-humio-core -o jsonpath='{"http://"}{.spec.host}{"\n"}'
# Password for user developer
kubectl -n humio-logging get secret developer-user-password -o jsonpath="{.data.password}" | base64 -d
```

## Demo: Log Anomaly Detection

### Define dynamic template to create resource groups for each service in Robot Shop

- From the main navigation, click `Application management`.
- From the Application management page, launch the template manager by clicking the rightmost icon `Group templates` in the upper right side corner. The page for creating an template is displayed.
- Click `Create a new template` and choose `Dynamic template` type to create. Here, we will ask Topology Mananger to create a group for each service in Robot Shop and where they're running so we can manage them independently and as part of an application. 
- Use the `template builder` to define the criteria for Sample App template. Enter the following parameters:

| Field                 | Value                                                 |
| --------------------- | ----------------------------------------------------- |
| Name                  | My Robot Shop Services
| Group Type            | compute
| Tag                   | robots
| Search for a resource to get started | Type `web` and you'll see a list of hits, choose `web service`, navigate using the right-click menus and use `Get neighbors` and choose `Pod`, then `Get neighbors` and choose `server`.

### Create application for Robot Shop

Creating an application involves selecting the groups of resources to compose the application.

- From the main navigation, click `Application management`.
- Click `Create application`. The page for creating an application is displayed.
- Select the groups of resources that you want to include in the application and click `Add to application`. You can also drag a group into the sidebar to add the group to the application. Selecting `robot-shop`  with `namespace` type and `cart`, `catalogue`, `dispatch`, `mongodb`, `mysql`, `payment`, `rabbitmq`, `ratings`, `redis`, `shipping`, `user`, `web` with `compute` type and add into application.
- Enter a name for the application, take `robot-shop-app` as example.
- (Optional) Set the application as a favorite application.
- Set an icon for the application. This icon is used to identify  the application resource type in the topology view of your applications.
- (Optional) Select any tags for classifying the application.  Tags can be used to identify similar applications and to distinguish  applications from other applications. To add a tag, you need to search  for the tag by name and then select the tag.
- (Optional) Remove any groups that you do not want in the application.
- Click `Create`. The application is added to the  list of all applications on the dashboard, and if selected, to the list  of favorite applications.

### Enable Humio live log and trigger applicaton story/alert

To enable Humio live log, in WAIOPS Web UI:

- From the `Data and tool integrations` page, click integration list in the Humio tile under Logs category
- Select the robot-shop sample app Humio integration  `humio-robot-shop`  in  list created by ansible playbook
- Change Mode to `Live data for continuous AI training and anomaly detection`, and enable `Data flow` On.

Wait 30 minutes to let log flow into.

### Inject 401 issue into ratings service in Robot Shop

To simulating HTTP issue, an istio VirtualService with HTTP RC 401 is created to simulate HTTP unauthorized issue on `ratings`:

```sh
echo "
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings-http-unauthorized
  namespace: robot-shop
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 401
        percentage:
          value: 100
    route:
    - destination:
        host: ratings
" | kubectl apply -f -        
```

In robot-shop web UI, in categories, click `Watson` and rate it to trigger the `401` issue. Repeat several for other robot.

### Verify log anomaly detecting in application

For application story, in defined application `robot-shop-app`,  stories can be found per detected abnormal logs.

For alert, drill down the story to alerts, and when select one, details will show.

For slack, from alert page, clicke right-top `Slack` to navigate slack console for those alerts.  
