---
layout: post
title: 'Deep Dive: GitOps with Flux v2 in Microsoft Azure Kubernetes Service (AKS)'
date: 2022-06-29 14:30:20 +0300
description: 'Deep Dive: GitOps with Flux v2 in Microsoft Azure Kubernetes Service (AKS)' # Add post description (optional)
img: 2022-04-29-flux.jpg # Add image post (optional)
fig-caption: Photo By Joe Ciciarelli on Unsplash # Add figcaption (optional)
tags: [ gitops, flux, azure, aks, k8s, containers, kubernetes, kustomize, helm ]
---

While the term "GitOps" has achieved almost mainstream use, confusion remains around what it is and the benefits it can bring. True, it involves continuous delivery (CD), but the way delivery is achieved as well as how CD interacts with workload operations, that is where GitOps gets really interesting.  

Part of GitOps is leveraging Git abstractions like branches, pull requests and approval flows to manage the operational process, but GitOps is far more than the last step of CI. 

Instead it is about leveraging the most fundamental element of Kubernetes, reconciliation, both for CD and to link CD with the reconcilers that are automating workload operations.

# 1. Introduction

![logo]({{site.baseurl}}/assets/img/2020-10-14-flux-logo.png){: style="float: left"}

At Microsoft Build 2022 earlier this year, Microsoft **[announced the general availability](https://techcommunity.microsoft.com/t5/azure-arc-blog/announcing-general-availability-for-gitops-with-flux-v2-in-azure/ba-p/3408051)** of GitOps with **[CNCF Flux v2](https://www.cncf.io/projects/flux/)** in AKS.  The value that customers get by having GitOps as a managed service goes along with the same value as with all the Azure extensions.  

The advantage is that you get the lifecycle of the agent associated with the service through the Azure API.

The reason why Microsoft wants to make it a first class citizen in Azure is to enable customers to be able to use GitOps for managing configuration and application deployment in Kubernetes.  
The Flux v2 extension is an Azure managed GitOps service where you get :
- lifecycle via Azure API
- monitor the install status of the Flux v2 agents
- Azure verifies the versions that are being installed
- automatic agent updates
- support

By introducing Flux, we are moving to a declarative model where everything is version controlled and deployed by Flux thereby reducing manual operations, improving deployment consistency and also bringing more compliance with regard to change management and application life cycle.

GitOps in Azure allows you to manage your cluster configuration and applications in all your cloud clusters in one place in a secure and efficient manner. 

![design]({{site.baseurl}}/assets/img/2022-06-29-design.png)
Image by Microsoft; **[GitOps Flux v2 configurations with AKS](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-gitops-flux2#flux-cluster-extension)**

In this blogpost we will walk through the setup and a demo of Flux v2 on Azure.

# 2. Azure managed GitOps service

Within each cluster, the Flux operator is configured to sync a Git repository on a frequent basis down to the cluster and apply the Kubernetes manifests.  Once you've done a pull request into the Git repository and it has been merged, the Flux operator will automatically pull those changes and apply them automatically.  

There is no need for any person or any tool to actually access the Kubernetes cluster and do the deployments in the cluster.

The agent that is running in the Kubernetes cluster is continuously watching the state of the cluster and comparing it to the state that was declared in the Git repository. If the cluster has drifted for any reason, the Flux agent will attempt to reconcile the cluster back to the declared state.  It acts as a desired state configuration operator.

When you look in detail to the Flux v2 implementation, we can identify the following parts:

**1) Cluster extension** - a way to install services in AKS like monitoring, machine learning, policies, security and now also GitOps with Flux.  By default you will get the following controllers:
- Source
- Kustomize
- Helm
- Notification
You have the option the install the image-automation or image-reflector controllers.  All these are installed in the flux-system namespace.

Flux v2 is a major update compared to Flux v1 as it has become Kubernetes native. It's based on controllers and Custom Resource Definitions (CRDs).  When you deploy a Flux v2 configuration resource against the cluster, you require to define a source.  This is a Git repository, a path and a branch that can be used to pull the manifest.  The Kustomize controller manages the deployment of the Kubernetes manifests.

**2) Flux configurations** are used to setup the configuration of the source you are going to be tracking and the Kustomization(s) that are going to be deployed to the cluster.  By default, you define a source and one or more  Kustomization(s).  A Flux configuration can be seen as one instance of a Git repository that is going to be deployed.

You can have one or many Flux configurations per cluster.  Each Flux configuration can get it's own namespace scope.

## 2.1 Multi-tenancy within a company

A typical GitOps scenario in the large company is where you have one Flux configuration for your Kubernetes admin team to lay down all the baseline configurations across all your Kubernetes clusters.  
Next, you can have other Flux configurations; one for each different application team. Or one for each different business group and the applications they need to deploy.  This configuration would point to their specific Git repository and deploy their specific application(s).

This enables the multi-tenancy and separation of concerns in Kubernetes between the cluster admin teams and the application teams.  The cluster admin teams setup the clusters, do the core configurations and then they will delegate out just namespaces within Kubernetes clusters for the different application teams.  The application teams focus on bringing business value through their applications.

In Flux v2 you can have dependencies between the Kustomizations.  Deploy everything what is **infra-related** first and once that is deployed start with deploying the necessary **application-related** manifests.  With this method, you can line up how and what should be deployed first and what should be deployed second.

# 3. IaC & GitOps configuration

We are using a Bicep template to install the Flux extension on our AKS cluster.  So we programmatically deploy the AKS cluster, deploy the Flux v2 extension and deploy the Flux configuration on top of them to kind of bootstrap a cluster with the applications we need.  

The deployment is executed as in [previous]({% post_url 2022-01-11-using-openid-tokens-with-github-actions-to-azure %}) blogpost using a Github workflow (**[001-deploy.yml](https://github.com/dewolfs/flux2-gitops-aks-iac/blob/main/.github/workflows/001-deploy.yml)**) with Open ID Connect.

The image below is showing the **[Bicep template](https://github.com/dewolfs/flux2-aks/blob/main/deploy/iac-gitops.bicep)**:
![bicep]({{site.baseurl}}/assets/img/2022-06-29-bicep.png)

- **1)** The **Flux extension** is deployed on our AKS cluster.
- **2)** The **Flux configuration** is scoped to our AKS cluster.
- **3)** The **Source** block is referencing our Apps Git repository.
- **4)** The **infra Kustomization** is located in the infra folder.
- **5)** The **apps-dev Kustomization** is located in the apps folder with a subfolder defining our DEV config.  It has a dependency on infra.
- **6)** The **apps-prd Kustomization** is located in the apps folder with a subfolder defining our PRD config.  It has a dependency on apps-dev.

Our **[Git repository](https://github.com/dewolfs/flux2-aks-apps)** contains 2 directories: apps and infra.
You can think of the apps directory as storage for all of the applications that need to be deployed to a cluster. You might need to deploy different types of an application.  You will have different configurations for that application.

In this case we are using the **[Kuard](https://github.com/kubernetes-up-and-running/kuard)** application that will bring up pod info and there are some overlays on top of this application that change things based on which environment it's running in:   
- dev: 1 replica
- prod: 5 replica's

You might have a different version for production. You might have a different hostname for your ingress.  These are things you can configure as overlay for the environment-specific scenarios.

From the infra side, we are laying down some basic infrastructure components (**[Prometheus](https://prometheus.io/)**) to be able to run an app on top of it.  These infra components potentially needs to be lay down prior to your applications being spun up because the applications might have a dependency on these things to exist.

## 3.1 GitOps Configuration

In the **GitOps** section in the Azure portal, you can see the Flux configurations assigned to this AKS cluster.  

![bootstrap1]({{site.baseurl}}/assets/img/2022-06-29-bootstrap.png)  

One of the nice aspects of Flux v2 is the **compliance** information you get out of the deployments.  The compliance information is pulled back to Azure and made available in the Azure portal.
The most important piece is the compliance column. The Flux v2 implementation waits until every single Kubernetes object has been spun up and the status has been returned. The compliance indicates the compliance of all the deployments that are declared through this configuration and it's source. 

The compliance piece can be used in Azure Alerts to tell you if the cluster is compliant with the declarations in the Git repository for this particular configuration.

All of the Kubernetes declarations for this configuration are applied at the cluster scope.  The compliance state is showing that the installation of the agent tree succeedded.  

![bootstrap2]({{site.baseurl}}/assets/img/2022-06-29-bootstrap2.png)

If we dig into the configuration, we can see the compliance state of all the configuration objects.  The configuration objects are all of the Flux objects that are in the Git repository. 
![bootstrap3]({{site.baseurl}}/assets/img/2022-06-29-bootstrap3.png)

The source page contains information such as the repository URL and the reference type which contain all the Flux v2 capabilities (Branch, Commit, release Tag, Semver range).  If you choose a private repository, you must specify an authentication type.
![bootstrap4]({{site.baseurl}}/assets/img/2022-06-29-bootstrap4.png)

The Kustomizations page contains the details about the different paths used in the configuration.  The last column indicates the dependencies.  In our example has the **apps-dev** Kustomization a dependency on the **infra** being installed.  A similar relation has been defined between **apps-prd** and **apps-dev**.  

The Prune functionality means if we delete a Kustomization or delete the config then Flux will clean up all of the resources that were lay down in that cluster based on this configuration.  

![bootstrap5]({{site.baseurl}}/assets/img/2022-06-29-bootstrap5.png)

# 4. Results

As a result, the following namespaces are created:

- flux-system: Holds the Flux extension controllers.
- cluster-config: Holds the Flux configuration objects.
- monitoring: Namespace for workloads described in manifests in the Git repository - **folder infra**.
- kuard: Namespace for workloads described in manifests in the Git repository - **folder apps**.

![gitops]({{site.baseurl}}/assets/img/2022-06-29-gitops.png)

Flux does not require a master cluster like ArgoCD does. It's all individual clusters and with this approach we make sure everything can work in that sort of disconnected state where basically the master is your Git repository and how to build on top of that within Azure to give you some of these capabilities like observability across groups of clusters and across applications that are deployed to multiple clusters.

# 5. Observability

To track the compliance state of all your clusters across all your subscriptions you can use Azure Graph Resource Explorer.

```
kubernetesconfigurationresources
| where type == "microsoft.kubernetesconfiguration/fluxconfigurations"
| where resourceGroup contains "rg-aks-001"
| where properties.gitRepository.url == "https://github.com/dewolfs/flux2-aks-apps.git"
| parse id with * "/providers/" provider "/" clusterType "/" *
| summarize count() by tostring(properties.complianceState), clusterType, provider
```
The above query lists the compliance state by cluster type.  We are interested in getting all the Flux configurations from a specific resource group.  With this query we want to understand the compliance state of the configurations by cluster type and parse out the provider and clustertype.

![graphexplorer]({{site.baseurl}}/assets/img/2022-06-29-graphexplorer.png)

# 6. Lifecycle management Flux

Upgrades are driven by the Azure extension platform across different cluster types.  The extension platform is currently growing.

There are two options to manage Azure extensions in terms of the upgrade path:  
- **1) Disabling auto-upgrade**: manual doing version upgrades by calling the api directly  
- **1) Enabling auto-upgrade**: auto-upgrade with no configurable timeframe.  It's driven from the extension owner side.

*The configuration we used in this post can be found on <https://github.com/dewolfs/flux2-gitops-aks-iac> & <https://github.com/dewolfs/flux2-gitops-aks-apps >.*