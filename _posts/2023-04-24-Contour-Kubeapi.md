---
title: "Kubeapi with Contour"
date: 2023-04-24
---

# Securely expose Kubeapi with Contour HTTPProxy

Many Kubernetes workload clusters are provisioned with a random Certificate Authority. This allows the Cluster to create the dependent certificates and keys, providing security among the various services.

This is described in detail in the [Kubernetes Documentation](https://kubernetes.io/docs/setup/best-practices/certificates/).

Thanks to [**Dodd Pfeffer**](https://www.linkedin.com/in/doddpfeffer/) for helping me use Contour!

## TL;DR

The details of the solution are provided in a Git repo, including a detailed readme and sample code.

[Check out the repo here](https://github.com/cgsamp/k8s-custom-api-cert)

## Validate Certificate Authority

TLS relies on the server providing a Certificate. This Certificate references a Certificate Authority - essentially, the "notary" that "vouches" that the certificate was provided correctly and the service providing the certificate is who they say they are.

How do you know to trust the Certificate Authority?

The client system (your laptop, for example) has a list of CAs it will trust. Some are so-called "root CAs" that are shipped with your computer or its software. Others are individually added. For instance, a company may have their own CA that they place on their employee's systems.

It can also be specifically referenced, or manually added to a truststore.

## Certificate Authority in kubeconfig

Most of the time, the Kubernetes-generated  random Certificate Authority is distributed into the client `kubeconfig` file. (Validation can also be skipped with the `insecure-skip-tls-verify` option, defeating the system entirely.)

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t...Self-Signed (Base64-encoded generated certificate authority)
    server: https://kubeapi-server:
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

## Certificate not provided

Putting the certificate into the kubeconfig file is convenient, but it requires trusting that process. If a malicious actor is able to substitute their own self-signed certificate, they could create a person-in-the-middle attack and monitor all communications, including stealing secrets and data!

The Public Key Infrastructure (PKI) system is designed to prevent this. There is a trusted set of Root Certificate Authorities that can be trusted to issue certificates or intermediate certificate authorities. The CA that is provided as part of the Certificate could be validated against these trusted CAs.

That interaction shows what happens when the CA is not provided, and the self-signed certificate is used. Trust is not established.
![Sequence diagram](/assets/mermaid-diagram-2023-04-24-143121.png)

## Solutions

### Provide Kubernetes an intermediate CA

It is possible to provide the Kubernetes bootstrap service with a Certificate Authority to create all it's internal certificates. However, in practice, CAs are not often distributed as they could represent a security risk if lost, stolen or abused. There are many nuances to consider, like for what purposes the CA could be used. [Kubernetes docs discuss in detail.](https://kubernetes.io/docs/tasks/tls/manual-rotation-of-ca-certificates/#rotate-the-ca-certificates-manually)

### Manually change certificates

[It is also possible to `ssh` into the control plane nodes and swap out certificates.](https://kubernetes.io/docs/setup/best-practices/certificates/#configure-certificates-manually) I don't want to do that, do you??

## Solution: Ingress

Kubernetes has in Ingress class that can be fulfilled by a number of providers, like NGINX or Contour - Contour is our choice today.

We can create or obtain our own Certificate, and have Contour present that to the client; we can then have Contour turn around and re-encrypt the traffic by accepting the self-signed certificate. Since this all happens within our cluster, we can keep the self-signed certificate scope within our cluster.

![sequence diagram](/assets/mermaid-diagram-2023-04-24-144439.png)

## Solution: Load balancer

It is also possible to use a load balancer to perform TLS Termination and Workload Encryption. The elegance of using Ingress is that information about the Node IPs does not need to be exposed outside the cluster, making the system more maintainable.
