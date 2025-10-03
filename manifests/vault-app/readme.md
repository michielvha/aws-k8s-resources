# Deploying vault inside kubernetes

Currently, using standalone, in prod use HA.

for testing, I am using etcd as a backend for vault. since we already have it running.

- [Use ETCD as backend](https://developer.hashicorp.com/vault/docs/configuration/storage/etcd)

we need to expose etcd as a service in the cluster for this
````shell
kubectl expose pod etcd-main-x86-prd-01.mvha.lan --name=etcd-service --port=2379 --target-port=2379 -n kube-system
````

## Storage Provider

Select a storage provider for your deployment
- [Storage Provider Docs](https://developer.hashicorp.com/vault/docs/configuration/storage)


https://developer.hashicorp.com/vault/docs/platform/k8s

- [Main deployment guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide)
- [Production setup](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide#configure-vault-helm-chart)
- https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-raft




https://developer.hashicorp.com/vault/docs/configuration/storage/raft

[Configure the helmchart](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide#configure-vault-helm-chart)

# Running in etcd

Yes, if you mount the following **etcd TLS certificate files** into your Vault deployment, you can authenticate securely with the etcd server. Here's what you need to do:

---

## **1️⃣ Required Files to Mount into Vault**
From your RKE2 etcd config, these are the required files:

- **CA certificate** → `/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt`
- **Client certificate** → `/var/lib/rancher/rke2/server/tls/etcd/server-client.crt`
- **Client key** → `/var/lib/rancher/rke2/server/tls/etcd/server-client.key`

---

## **2️⃣ Update Vault's Configuration**
Modify your `vault.hcl` file or `values.yaml` (if using Helm):

```hcl
storage "etcd" {
  address       = "https://192.168.1.241:2379"
  etcd_api      = "v3"
  ha_enabled    = "true"
  tls_ca_file   = "/etc/vault/etcd/server-ca.crt"
  tls_cert_file = "/etc/vault/etcd/server-client.crt"
  tls_key_file  = "/etc/vault/etcd/server-client.key"
}
```
- Ensure the `address` points to your etcd server (`192.168.1.241:2379`).
- Set the correct paths inside the Vault container (`/etc/vault/etcd/`).

---

## **3️⃣ Mount the TLS Files into the Vault StatefulSet**
If you're using Kubernetes **manifests**, update the Vault `Deployment`:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vault
  namespace: vault
spec:
  template:
    spec:
      volumes:
        - name: etcd-certs
          hostPath:
            path: /var/lib/rancher/rke2/server/tls/etcd
            type: Directory
      containers:
        - name: vault
          volumeMounts:
            - name: etcd-certs
              mountPath: /etc/vault/etcd
              readOnly: true
```

If using **Helm**, set this in `values.yaml`:
```yaml
server:
  volumes:
  - name: etcd-certs
    hostPath:
      path: /var/lib/rancher/rke2/server/tls/etcd
      type: Directory
    
  volumeMounts:
  - name: etcd-certs
    mountPath: /etc/vault/etcd
    readOnly: true
```
Then upgrade Vault:
```sh
helm upgrade --install vault hashicorp/vault -f values.yaml -n vault
```

---

## **4️⃣ Verify Connectivity**
Inside the Vault pod, test etcd access:
```sh
kubectl exec -it vault-0 -n vault -- wget --no-check-certificate -O- https://192.168.1.241:2379/version
```
If this works, restart Vault:
```sh
kubectl rollout restart deployment vault -n vault
```

---

## **TL;DR: Steps to Fix**
✅ **Mount these etcd certs into Vault**  
✅ **Update `vault.hcl` to use TLS authentication**  
✅ **Restart Vault and check logs**

After this, Vault should be able to authenticate with etcd. 🚀 Let me know if you need any help debugging!

---
# Different versions
HashiCorp Vault offers multiple deployment methods on Kubernetes to accommodate various use cases and environments. The primary deployment configurations include:

1. **Development (Dev) Mode**: This mode deploys a single in-memory Vault server, ideal for testing and development purposes. It is not recommended for production environments due to its lack of persistence and high availability.

2. **Standalone Mode**: In this default configuration, a single Vault server uses a file storage backend, persisting data to a volume. While suitable for basic setups, it lacks the resilience required for production workloads.

3. **High-Availability (HA) Mode**: This setup involves a cluster of Vault servers utilizing an HA storage backend, such as Consul or Vault's Integrated Storage (Raft). HA mode ensures that Vault remains available even if some nodes fail, making it suitable for production environments.

4. **External Mode**: In this scenario, Vault runs outside the Kubernetes cluster, and Kubernetes workloads authenticate and retrieve secrets from this external Vault instance. This approach is beneficial when a centralized Vault service is preferred or when integrating multiple Kubernetes clusters.

The [Vault Helm chart](https://developer.hashicorp.com/vault/docs/platform/k8s/helm) is the recommended tool for deploying Vault on Kubernetes. It allows users to specify the desired deployment mode and configure various aspects of the Vault setup. By default, the Helm chart deploys Vault in standalone mode, but it can be customized to deploy in HA mode with Integrated Storage by adjusting the configuration parameters. citeturn0search8

For detailed guidance on deploying Vault with Integrated Storage on Kubernetes, refer to the [Vault on Kubernetes deployment guide](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide). This tutorial provides step-by-step instructions to configure the Vault Helm chart for a production-ready HA deployment. citeturn0search0

By understanding these deployment methods and utilizing the Helm chart, you can tailor your Vault deployment to meet your organization's specific requirements and ensure a secure and resilient secrets management solution on Kubernetes. 