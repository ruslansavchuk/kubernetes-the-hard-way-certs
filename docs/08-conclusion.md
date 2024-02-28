# Kubeadm certificates mapping
## ETCD
etcd-cluster-to-client-server -> etcd/server
etcd-peer -> etcd/peer
etcd-ca-peer -> etcd/ca
etcd-ca-cluster-to-client -> etcd/ca

## api-server
etcd-cluster-to-client-client -> apiserver-etcd-client
kubernetes-kubelet-server -> apiserver-kubelet-client
kubernetes-api-server -> apiserver
service-account -> sa
kubernetes-ca-api -> ca
kubernetes-ca-kubelet -> ""

## kubelet

## controller manager
kube-controller-manager -> values from controller-manager.conf

## scheduler
kube-scheduler -> values from scheduler.conf

## kube-proxy


![image](schema.png "Container runtime")
*