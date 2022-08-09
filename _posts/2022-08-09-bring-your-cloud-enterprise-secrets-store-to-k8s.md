---
layout: post
title: 'Bring your own cloud Enterprise secrets store (Azure Key Vault) to Kubernetes'
date: 2022-08-09 03:30:20 +0300
description: 'csi-driver' # Add post description (optional)
img: 2022-08-09-csi.jpg # Add image post (optional)
fig-caption: Photo By Arto Marttinen on Unsplash # Add figcaption (optional)
tags: [ azure, aks, k8s, containers, kubernetes, secrets, keyvault, csi ]
---

Applications running on Kubernetes require access to sensitive information (passwords, SSH keys and authentication tokens). But how do you configure your applications when the source of truth for these secrets is an external secret store? What if you need to store, retrieve and perform zero touch rotation of these secrets securely?

# 1. Introduction

Why are we talking about external secret storage when Kubernetes has one built-in?  Kubernetes secrets may not meet your data-at-rest encryption requirements. When you create a secret in kubernetes, it is stored in etcd and it's base64 encoded but not encrypted.

There are KMS providers that enable secret data encryption-at-rest.

Dependening on your organization, you may have already standardized on a third-party secret solution and need to use that.
Your external secret solution may have some sort of secret rotation that you are looking to leverage.

There are a few options to consume external secrets:
1. You might look at modifying your application to fetch the secret from external API directly using the SDKs provided by the external secret store provider.
2. You might copy the secrets and sync them as Kubernetes secrets via a simple controller.
3. You could use a side-car to fetch and write secrets. The sidecar is injected using a mutating webhook.
4. The secrets store CSI driver that uses a Container Storage Interface(CSI) specification.

![logo]({{site.baseurl}}/assets/img/2022-08-09-csi-logo.png){: style="float: left"} The **[Container Storage Interface (CSI) driver](https://kubernetes-csi.github.io/)** allows Kubernetes to mount multiple secrets, keys and certificates stored in enterprise grade external store into a pod as a volume.  Once the volume is attached, the data in it is mounted into the container's temporary file system (tmpfs).

Features:
- It has a familiar filesystem mount experience to compute workloads.  
- It's pluggable and it supports multiple external secret providers without the application having to be modified.  
- It can load new values of secrets throughout the life cycle of the pod.  
- It can sync the mounted content as Kubernetes secret for further compatibility with existing deployments.  

The driver supports multiple external secret providers.  The supported providers which are available today with the secret store csi driver are:  
- Azure Key Vault
- Google Secret Manager
- HashiCorp Vault
- AWS Secrets Manager

In this blogpost we will walk through the setup and configuration of the CSI driver in collaboration with Azure Key Vault.

# 2. How it works

The image below gives an overview of a steps involved in the process.

![design]({{site.baseurl}}/assets/img/2022-08-09-csi-design.png)  
Image by **[Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html)**

- **1)** A driver is installed as a daemon set on each node in the cluster. In addition to the driver being deployed, there need to be a provider specific daemon set deployed on every single node.
- **2)** First when a pod is created through the Kubernetes API, it gets scheduled onto a particular node based on scheduling decisions.  The kubelet process on the node looks at the pod spec and sees there is a volume mount request and referencing a particular CSI driver.
- **3)** The kubelet issues an RPC call to the CSI driver provided in the CSI volume.
- **4)** The first thing that the CSI driver does, is it mounts the tempfs to the pod. Then it issues an workload identity or any other configuration that is tied to the workload pod.
- **5)** the provider fetches the secrets from the external secrets store and it sends the content back to the driver as part of the RPC response.  The driver then writes the secrets into the file system and at this point the volume is successfully mounted and the pod starts up running.

# 3. Kubernetes Cluster bootstrap

The provisioning of the AKS cluster is executed via **[GitHub Actions](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/.github/workflows/001-infra-deploy.yml)** using the Azure CLI.
We continue to adopt the best practice as discussed in a previous post *[Using OpenID Connect (OIDC) tokens with GitHub Actions and Azure]({% post_url 2022-01-11-using-openid-tokens-with-github-actions-to-azure %})*.

By enabling the CSI driver Add-on using the Azure CLI, the CSI driver integrates with our AKS cluster and it provides a fully supported installation of the CSI driver.

```
az aks create \
  --resource-group env.RESOURCE_GROUP \
  --name env.AKS_NAME \
  --location env.LOCATION \
  --node-count 1 \
  --node-vm-size Standard_D2s_v5 \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval=1m \
  --enable-managed-identity \
  --generate-ssh-keys
```

![csipods]({{site.baseurl}}/assets/img/2022-08-09-csi-pods.png)

When the **az aks create** CLI completes, besides the Kubernetes cluster, we also get an user-assigned managed identity, named **azurekeyvaultsecretsprovider-***.  

![csipods]({{site.baseurl}}/assets/img/2022-08-09-managed-identity.png)

This user-assigned managed identity will be used to pull secrets from our Azure Key Vault.  A task in our GitHub workflow will assign the least privileged Azure RBAC built-in role 'Key Vault Secrets User'.

![csipods]({{site.baseurl}}/assets/img/2022-08-09-kv.png)

# 4. Kubernetes Manifests

The configuration of the CSI driver and the Azure Key Vault provider are defined in Kubernetes manifest files.

**Secret mounted as volume:**
- **[manifests/01-secretProviderClass.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/01-secretProviderClass.yml)**
- **[manifests/02-deployment.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/02-deployment.yml)**

**Secret synced as Kubernetes secret & environment variable:**
- **[manifests/10-secretProviderClass_secretObjects.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/10-secretProviderClass_secretObjects.yml)**
- **[manifests/11-deployment.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/11-deployment.yml)**

## 4.1 Mount secret in volume

We explained how the driver works, next we will discuss the YAML files required for the configuration.

![class+pod]({{site.baseurl}}/assets/img/2022-08-09-class&pod.png)

The **[01-secretProviderClass.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/01-secretProviderClass.yml)** manifest file has a spec that contains the provider (Azure) that will be used.  Access to our Azure Key Vault is handled by our user-assigned managed identity (Client ID azurekeyvaultsecretsprovider-*).  Offcourse the Azure Key Vault name is specified together with the secret objects.

The **[02-deployment.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/02-deployment.yml)** manifest describes our deployment, we have the mount path defined which is the path inside the container (/mnt/secrets-store).  If we look at the volume, we can see it's a CSI volume and not a native Kubernetes secret volume.  The driver name in the csi volume is what tells the kubelet to use the secret store CSI for this particular volume.
The **secretProviderClass** in the **volumeAttributes** is a namespace Kubernetes custom resource.  It is used to provide driver configurations and provide a specific parameters for the secret store CSI driver.

Apply both manifest files and check the content of our secret in the specified volume mount path:

```
export POD_NAME=$(kubectl get pods -l "app=nginx" -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it -n default $POD_NAME -- cat /mnt/secrets-store/db-user
```

## 4.2 Sync secret as Kubernetes secret

The driver also supports synchronizing the secrets from the external secret store as Kubernetes secret. The CSI driver provides an optional feature to sync the mounted content from the pod as a Kubernetes secret.  A common usage of this feature is to store a TLS certificate in the external secret store and have the CSI driver sync it as a Kubernetes TLS secret and then that can be used with an Ingress controller.  

![class+pod+secret]({{site.baseurl}}/assets/img/2022-08-09-class&pod&secret.png)

As you can see in the **[10-secretProviderClass_secretObjects.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/10-secretProviderClass_secretObjects.yml)** manifest, there is an optional **secretObjects** field which is used to indicate that the mounted content also needs to be synced as Kubernetes secret.  In the **[11-deployment.yml](https://github.com/dewolfs/secrets-store-csi-driver-azure-aks/blob/main/manifests/11-deployment.yml)** manifest, the Kubernetes secret is referenced in the container spec together with the key.

The Kubernetes secret is base64-encoded with the name as referenced in the container spec.

![secret-k8s]({{site.baseurl}}/assets/img/2022-08-09-secrets-k8s.png)

## 4.3 Kubernetes environment variable

In addition to the synced Kubernetes secrets, they can also be referenced in the pod spec to be set as an environment variable if required.  

```
export POD_NAME=$(kubectl get pods -l "app=nginx" -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it -n default $POD_NAME -- env
```
![secret-env]({{site.baseurl}}/assets/img/2022-08-09-secrets-env.png)

# 5. Secret rotation  

A general best practice is to periodically rotate your secrets.  Your workload running on a Kubernetes cluster should get the new values of the secret whenever it changes.  The driver supports automatic rotation by periodically reissuing the RPC calls to the provider to refresh the mount.  Once the rotation is successfully done in the mount, the driver will also emit a Kubernetes event to indicate that a rotation is successfully completed.

If the mounted content was also synced as a Kubernetes secret then the driver will also go and update the values in the Kubernetes secret.  
![rotate-log]({{site.baseurl}}/assets/img/2022-08-09-rotate-log.png)

## 5.1 Secret versioning

When we create our AKS cluster, we have enabled secret rotation together with the poll interval.
```
  --enable-secret-rotation
  --rotation-poll-interval=1m
```  

*How do we know which version of the secret that you get from our external secret store is currently being used by our pod?*

The **secretProviderClassPodStatus** is a namespace-specific Kubernetes custom resource that is created and managed by the secret store CSI driver to track the binding between a pod and a secretProviderClass.  The naming convention will always be **pod name-namespace-secretproviderclass name**.

```
export SECRETPROVIDERCLASS=$(kubectl get secretProviderClassPodStatus)
kubectl get secretProviderClassPodStatus $SECRETPROVIDERCLASS -o yaml
```
![secretProviderClassPodSTatus]({{site.baseurl}}/assets/img/2022-08-09-secretProviderClassPodStatus.png)  
This resource contains details about the current object versions that have been loaded into the pod.  
Whenever the secret changes in the external secret provider, the latest version of the secret will be made available in the pod.

The version-GUID of the secret in the Azure Key Vault is identical to the value we get in **status.objects.id.version**.

![secretProviderClassPodSTatus]({{site.baseurl}}/assets/img/2022-08-09-secrets-version.png)  

*The configuration we used in this post can be found on <https://github.com/dewolfs/secrets-store-csi-driver-azure-aks>.*