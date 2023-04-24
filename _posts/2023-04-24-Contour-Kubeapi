---
title: "Kubeapi with Contour"
date: 2023-04-24
---


# Securely expose Kubeapi with Contour HTTPProxy

Many Kubernetes workload clusters are provisioned with a random Certificate Authority. This allows the Cluster to create the dependent certificates and keys, providing security among the various services.

This is described in detail in the [Kubernetes Documentation](https://kubernetes.io/docs/setup/best-practices/certificates/).

## Validate Certificate Authority

### Certificate Authority in kubeconfig

Most of the time, this random Certificate Authority is distributed into the client `kubeconfig` file. 

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t...Self-Signed (Base64-encoded generated certificate authority)
    server: https://kubeapi-server:6443
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    user: my-cluster-user
  name: my-cluster
current-context: my-cluster
kind: Config
preferences: {}
users:
- name: my-cluster-user
  user:
    token: eyJr...User-Token (Base64-encoded user token provided by Kubernetes admin)
```

When an operator attempts to access the cluster, for instance with `kubectl get nodes`, the following workflow takes place.


```
➜ ✗ curl https://kubeapi-server:6443/api/v1/nodes -vv
*   Trying ip-address:6443...
* Connected to ip-address (ip-address) port 6443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* [CONN-0-0][CF-SSL] (304) (OUT), TLS handshake, Client hello (1):
* [CONN-0-0][CF-SSL] (304) (IN), TLS handshake, Server hello (2):
* [CONN-0-0][CF-SSL] (304) (IN), TLS handshake, Unknown (8):
* [CONN-0-0][CF-SSL] (304) (IN), TLS handshake, Certificate (11):
* [CONN-0-0][CF-SSL] (304) (IN), TLS handshake, CERT verify (15):
* [CONN-0-0][CF-SSL] (304) (IN), TLS handshake, Finished (20):
* [CONN-0-0][CF-SSL] (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384
* ALPN: server accepted h2
```
SUCCESS!!!

![Sequence diagram](/assets/mermaid-diagram-2023-04-24-141910.png)

### Certificate not provided

This requires additional trust on part of the client, and a chain of custody over the self-signed certificate. If a malicious actor is able to substitute their own self-signed certificate, they could create a person-in-the-middle attack and monitor all communications, including stealing secrets and data!

The Public Key Infrastructure (PKI) system is designed to prevent this. There is a trusted set of Root Certificate Authorities that can be trusted to issue certificates or intermediate certificate authorities. The CA that is provided as part of the Certificate could be validated against these trusted CAs.

That interaction would happen as follows:
![Sequence diagram](/assets/mermaid-diagram-2023-04-24-143121.png)

### Modify Kubernetes certificates

It is possible, as shown in the documentation, to provide the Kubernetes bootstrap service with a Certificate Authority to create all it's internal certificates. However, in practice, CAs are not often distributed as they could represent a security risk if lost, stolen or abused.

It is also possible to `ssh` into the control plane nodes and swap out certificates. I don't want to do that, do you??

## Solution: Ingress!

Kubernetes has in Ingress class that can be fulfilled by a number of providers, like NGINX or Contour - Contour is our choice today.

We can create or obtain our own Certificate, and have Contour present that to the client; we can then have Contour turn around and re-encrypt the traffic by accepting the self-signed certificate. Since this all happens within our cluster, we can keep the self-signed certificate scope within our cluster.

![sequence diagram](/assets/mermaid-diagram-2023-04-24-144439.png)


# Deploying

## Install Contour

To quickly install OSS Contour, access the kubernetes cluster with a user having sufficent rights.

```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

* In a newly installed TKGs cluster, this role mapping may be needed:
```
kubectl create rolebinding rolebinding-default-privileged-sa-ns_default --namespace=projectcontour --clusterrole=psp:vmware-system-privileged --group=system:serviceaccounts

```

## Get Contour / Envoy IP address

```
kubectl get service/envoy -n projectcontour -o json | jq -r '.status.loadBalancer.ingress[0].ip'
```

## Create the Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: csamp-tanzu-wildcard
data:
  tls.crt: [base64-encoded certificate]
  tls.key: [base64-encoded private key] 
type: kubernetes.io/tls

```

## Create the Ingress

Actually, we are using Contour's HTTPProxy, as it allows upstream TLS.

```
kind: HTTPProxy
apiVersion: projectcontour.io/v1
metadata:
  name: kubeapi-proxy
  namespace: default
spec:
  virtualhost:
    fqdn: k8s.csamp-tanzu.com
    tls:
      secretName: csamp-tanzu-wildcard
  routes:
    - services:
        - name: kubernetes
          port: 443
          protocol: tls
```

## Create the DNS entry

Use the DNS provider to map the address for which you have a certificate to the envoy IP address.

## Create a new kubeconfig file

```
apiVersion: v1
clusters:
- cluster:
    server: https://k8s.csamp-tanzu.com
  name: remote-tkgs-workload-cluster-1
contexts:
- context:
    cluster: remote-tkgs-workload-cluster-1
    user: third-party
  name: remote-tkgs-workload-cluster-1
current-context: remote-tkgs-workload-cluster-1
kind: Config
preferences: {}
users:
- name: third-party
  user:
    token: [User-token]

```

## Connect to the Kubernetes api!

```
✗ kubectl --kubeconfig csamp-tanzu-kubeconfig.yaml get nodes    
NAME                                                              STATUS   ROLES                  AGE     VERSION
tkgs-workload-cluster-1-control-plane-fdcnt                       Ready    control-plane,master   4h32m   v1.23.8+vmware.3
tkgs-workload-cluster-1-control-plane-s6nhj                       Ready    control-plane,master   4h37m   v1.23.8+vmware.3
tkgs-workload-cluster-1-control-plane-zjsjx                       Ready    control-plane,master   4h35m   v1.23.8+vmware.3
tkgs-workload-cluster-1-worker-nodepool-a1-wdvbn-65bb4c9796wtvj   Ready    <none>                 4h35m   v1.23.8+vmware.3
tkgs-workload-cluster-1-worker-nodepool-a1-wdvbn-65bb4c979frzb8   Ready    <none>                 4h35m   v1.23.8+vmware.3
tkgs-workload-cluster-1-worker-nodepool-a1-wdvbn-65bb4c979w6wjw   Ready    <none>                 4h35m   v1.23.8+vmware.3
```
