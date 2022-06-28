## Configure vSphere cloud provider and vSphere container storage interface for k8s cluster

### Prerequisites


1- vSphere requirements 
  (vSphere 6.7U3 or later)

2- Install VMwareTools

3- Install govc

```bash
curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" | tar -C /usr/local/bin -xvzf - govc

```

4- Install jq

```bash
apt install jq
```

5- Set disk.EnableUUID=1 on all nodes

```bash
export GOVC_INSECURE=1
export GOVC_URL='https://172.16.30.100'
export GOVC_USERNAME=''
export GOVC_PASSWORD=''


govc vm.change -vm '/Dev Datacenter/vm/KUBE/RKM1' -e="disk.enableUUID=1"
govc vm.change -vm '/Dev Datacenter/vm/KUBE/RKM2' -e="disk.enableUUID=1"
govc vm.change -vm '/Dev Datacenter/vm/KUBE/RKW1' -e="disk.enableUUID=1"
govc vm.change -vm '/Dev Datacenter/vm/KUBE/RKW2' -e="disk.enableUUID=1"

```

6- kubectl patch node

```bash
cd rke/vsphere
./govc
```

7- Upgrade Virtual Machine Hardware
   (VM Hardware should be at version 15 or higher)



## Install the vSphere Cloud Provider Interface


1- Create a CPI configMap


```bash

# Global properties in this section will be used for all specified vCenters unless overriden in VirtualCenter section.
global:
  port: 443
  # set insecureFlag to true if the vCenter uses a self-signed cert
  insecureFlag: true
  # settings for using k8s secret
  secretName: cpi-global-secret
  secretNamespace: kube-system

# vcenter section
vcenter:
  tenant-org:
    server: 
    datacenters:
      - Datacenter
    secretName: cpi-secret
    secretNamespace: kube-system

# labels for regions and zones
labels:
  region: rke-region
  zone: rke-zone

```



```bash

cd /etc/kubernetes
kubectl create configmap cloud-config --from-file=vsphere.conf --namespace=kube-system
kubectl get configmap cloud-config --namespace=kube-system

```

2- Create a CPI secret


```bash

apiVersion: v1
kind: Secret
metadata:
  name: cpi-secret
  namespace: kube-system
stringData:
  <vcenter-ip>.username: ""
  <vcenter-ip>.password: ""

```


```bash

kubectl create -f cpi-secret.yaml
kubectl get secret cpi-secret --namespace=kube-system

```

3- Check that all nodes are tainted

```bash

kubectl taint nodes rkm1 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl taint nodes rkm2 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl taint nodes rkw1 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl taint nodes rkw2 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl describe nodes | egrep "Taints:|Name:"

```


4- Deploy the CPI manifests

```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl apply -f https://github.com/kubernetes/cloud-provider-vsphere/raw/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml

```


5- Verify that the CPI has been successfully deployed

```bash

kubectl get pods --namespace=kube-system

```


6- Check that all nodes are untainted

```bash

kubectl describe nodes | egrep "Taints:|Name:"

```

## Install vSphere Container Storage Interface 


1- Check that all nodes are tainted


```bash

kubectl taint nodes rkm1 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl taint nodes k8s-worker-1 node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
kubectl describe nodes | egrep "Taints:|Name:"

```


2- Identify the Kubernetes major version. For example, if the major version is 1.22.x, then run the following.


```bash

VERSION=1.22
wget https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/release-$VERSION/releases/v$VERSION/vsphere-cloud-controller-manager.yaml

```

3- Create a vsphere-cloud-config configmap of vSphere configuration
    Modify the vsphere-cloud-controller-manager.yaml


```bash

apiVersion: v1
kind: Secret
metadata:
  name: vsphere-cloud-secret
  labels:
    vsphere-cpi-infra: secret
    component: cloud-controller-manager
  namespace: kube-system
  # NOTE: this is just an example configuration, update with real values based on your environment
stringData:
  172.16.30.100.username: "vcenter-user"
  172.16.30.100.password: "vcenter-pass"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: vsphere-cloud-config
  labels:
    vsphere-cpi-infra: config
    component: cloud-controller-manager
  namespace: kube-system
data:
  # NOTE: this is just an example configuration, update with real values based on your environment
  vsphere.conf: |
    # Global properties in this section will be used for all specified vCenters unless overriden in VirtualCenter section.
    global:
      port: 443
      # set insecureFlag to true if the vCenter uses a self-signed cert
      insecureFlag: true
      # settings for using k8s secret
      secretName: vsphere-cloud-secret
      secretNamespace: kube-system

    # vcenter section
    vcenter:
      my-vc-name:
        server: 10.185.0.89
        user: Administrator@vsphere.local
        password: Admin!23
        datacenters:
          - VSAN-DC 


kubectl apply -f vsphere-cloud-controller-manager.yaml
rm vsphere-cloud-controller-manager.yaml

```

4- Create vmware-system-csi Namespace


```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/vsphere-csi-driver/v2.5.1/manifests/vanilla/namespace.yaml

```

5- Taint Kubernetes Control Plane Node


```bash

kubectl taint nodes rkm1 node-role.kubernetes.io/controlplane=:NoSchedule
kubectl describe nodes | egrep "Taints:|Name:"

```


6- Create a Kubernetes Secret


```bash

cat /etc/kubernetes/csi-vsphere.conf

[Global]
cluster-id = "<cluster-id>"
cluster-distribution = "<cluster-distribution>"
ca-file = "" # optional, use with insecure-flag set to false

[NetPermissions "A"]
ips = "*"
permissions = "READ_WRITE"
rootsquash = false

[VirtualCenter "192.168.100.25"]
insecure-flag = "true"
user = ""
password = ""
port = "443"
datacenters = ""
targetvSANFileShareDatastoreURLs = "" # Optional


kubectl create secret generic vsphere-config-secret --from-file=csi-vsphere.conf --namespace=vmware-system-csi
kubectl get secret vsphere-config-secret --namespace=vmware-system-csi
rm csi-vsphere.conf

```


7- Deploy vSphere Container Storage Plug-in



```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/vsphere-csi-driver/v2.5.1/manifests/vanilla/vsphere-csi-driver.yaml

kubectl get deployment --namespace=vmware-system-csi
kubectl describe csidrivers
kubectl get CSINode

```



### References


#### Cloud Provider Interface (CPI)
- https://github.com/kubernetes/cloud-provider-vsphere/blob/master/docs/book/cloud_provider_interface.md

#### Container Storage Interface (CSI)
- https://github.com/container-storage-interface/spec/blob/master/spec.md


#### Other docs
- https://cloud-provider-vsphere.sigs.k8s.io/tutorials/kubernetes-on-vsphere-with-kubeadm.html

- https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/index.html
