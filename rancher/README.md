## Setup kubernetes cluster with RKE

1- Add the Helm chart repository

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

```

2- Install cert-manager

```bash

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.8.0 \
  --set installCRDs=true

```

3- Create a Namespace for Rancher

```bash
kubectl create namespace cattle-system
```

4- Install Rancher with Helm

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=a.basirat@omid.ir \
  --set letsEncrypt.ingress.class=nginx
```

### References

- https://rancher.com/docs/rancher/v2.6/en/overview/
