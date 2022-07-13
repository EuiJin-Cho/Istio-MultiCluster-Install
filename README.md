# Istio-MultiCluster-Install

## Kubernetes install - watch kubernetes-installation and source kubeadm_installation.sh

### On Master Node
kubernetes init
```
kubeadm init --config kubeadm.conf

```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

install CNI
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

install cloud-provider-openstack
```
git clone https://github.com/kubernetes/cloud-provider-openstack.git

```

create secret
```
kubectl create secret -n kube-system generic cloud-config --from-file=cloud.conf

```

Create RBAC resources and openstack-cloud-controller-manager deamonset
```
kubectl apply -f cloud-provider-openstack/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl apply -f cloud-provider-openstack/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml

```

change line45 to your cluster name
```
kubectl apply -f openstack-cloud-controller-manager-ds.yaml

```

### On Worker Node
```
kubeadm join --config worker.yaml

```


## Istio Install Multi-Primary on different networks

### Both Cluster

Change your kubeconfig to configure multi cluster
```
vim ~/.kube/config

```

Set the environment Variables
```
export CTX_CLUSTER1=cluster1
export CTX_CLUSTER2=cluster2
export CTX_CLUSTER3=cluster3

```

```
cd ~
```

Download istio
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.6 sh -
cd istio-1.12.6/
export PATH=$PWD/bin:$PATH
kubectl label namespace default istio-injection=enabled

```


Make cacerts
```
export ISTIO_VER=1.12.6
export WORKDIR=~/
kubectl create namespace istio-system
kubectl create secret generic cacerts -n istio-system \
  --from-file=${WORKDIR}/istio-$ISTIO_VER/samples/certs/ca-cert.pem \
  --from-file=${WORKDIR}/istio-$ISTIO_VER/samples/certs/ca-key.pem \
  --from-file=${WORKDIR}/istio-$ISTIO_VER/samples/certs/root-cert.pem \
  --from-file=${WORKDIR}/istio-$ISTIO_VER/samples/certs/cert-chain.pem
  
```

### On cluster1
#### Set cluster1
Set the cluster’s network
```
kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2

```

Create the Istio configuration for cluster1
```
cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF

```

Apply the configuration to cluster1
```
istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml

```

Install the east-west gateway in cluster1
```
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster1 --network network1 | \
    istioctl --context="${CTX_CLUSTER1}" install -y -f -
    
```

Expose services in cluster1
```
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
    
```


#### Set cluster2

Create the Istio configuration for cluster2
```
cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
EOF

```

Apply the configuration to cluster2
```
istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml

```

Install the east-west gateway in cluster2
```
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster2 --network network2 | \
    istioctl --context="${CTX_CLUSTER2}" install -y -f -

```

Expose services in cluster2
```
kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml

```

#### Endpoint Discovery on cluster1

Enable Endpoint Discovery
Install a remote secret in cluster2 that provides access to cluster1’s API server
```
istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"

```
Install a remote secret in cluster1 that provides access to cluster2’s API server
```
istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"

```




