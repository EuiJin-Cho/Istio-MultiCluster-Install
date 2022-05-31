# Istio-MultiCluster-Install

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
