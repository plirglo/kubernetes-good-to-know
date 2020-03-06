# Certificates renewal in Kubernetes v1.13

Following set of commands allows to renew expired certificates in K8s v1.13. Renewing certs in later versions is different.
Kubernetes installed using `kubeadm` has self-signed set of certificates with validity of **1 year**. It is strongly recomennded to upgrade Kubernetes versions as often as it is possible  but more often than once per year.

## Execute on K8s master

### Backup /etc/kubernetes and /var/lib/kubelet

### Clean certs directories

```shell
sudo rm /etc/kubernetes/pki/*
sudo rm /etc/kubernetes/*.conf
```

Directory `/etc/kubernetes/pki/` should not have files (can have some dirs).
Directory `/etc/kubernetes/` should not have any `.conf` files.

### Renew certs using kubeadm

```shell
sudo kubeadm alpha certs renew all
K8S_IP=$(kubectl config view -o jsonpath={.clusters[0].cluster.server} | cut -d/ -f3 | cut -d: -f1)
sudo kubeadm init phase certs all --apiserver-advertise-address $K8S_IP
sudo kubeadm init phase kubeconfig all --apiserver-advertise-address $K8S_IP
```

### Renew configs

```shell
sudo \cp -arf /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo  chmod 777 $HOME/.kube/config
```

### Stop services

```shell
sudo systemctl stop kubelet
sudo systemctl stop docker
```

### Remove kubelet certificates and cache - it will be recreated when started again

```shell
sudo rm /var/lib/kubelet/pki/*
Sudo rm -rf /root/.kube/*
```

Make sure `/var/lib/kubelet/pki/` and `/root/.kube/` are empty.

### Start services

```shell
sudo systemctl start docker
sudo systemctl start kubelet
```
*Validate*
Now, `kubectl get nodes` should respond with list of node in `Not Ready` state. If it shows `Ready`, wait a moment - it needs time to find out that nodes are not ready. 

### For each node name generate `kubelet.conf`

```shell
sudo kubeadm alpha kubeconfig user --org system:nodes --client-name system:node:`NODE_NAME` > kubelet-`NODE_NAME`.conf
```

### Create and note new token for Nodes, it will be valid for 24h

```shell
sudo kubeadm token create
```

## Execute on each NODE

### Backup /etc/kubernetes and /var/lib/kubelet on node

### Stop Node's services

```shell
sudo systemctl stop kubelet
sudo systemctl stop docker
```

### Copy `ca.crt` from Master machine

Copy from **master** `/etc/kubernetes/pki/ca.crt` to Node's home directory (`~`)

### Make `ca.crt` be owned by root and replace existing ca cert on Node machine. 

```shell
sudo chown root:root ca.crt
sudo mv ca.crt /etc/kubernetes/pki/
```

### Clean Node's kubelet certs directories

```shell
sudo rm /var/lib/kubelet/pki/*
```

Directory `/var/lib/kubelet/pki/` should not have files.

### Copy kubelet.conf file generated on Master for current node (kubelet-`NODE_NAME`.conf)

### Rename it to have `kubelet.conf` name and make it root owned

```shell
mv kubelet-`NODE_NAME`.conf kubelet.conf
sudo chown root:root kubelet.conf
sudo chmod 600 kubelet.conf
```

### Move kubelet.conf to `/var/lib/kubelet/pki/`

```shell
sudo mv kubelet.conf /var/lib/kubelet/pki/
```

### Update bootstrap token (the one you noted on `token create` command)

```shell
new_token=`NEW_TOKEN_GENERATED_ON_MASTER`
sudo sed -i "s/token: .*/token: $new_token/" /etc/kubernetes/bootstrap-kubelet.conf
```

### Reboot worker node

```shell
sudo reboot
```

Do the same list of steps for all nodes.

## Execute on Master

### Delete and create service accounts that need to be recreated.

```shell
kubectl delete sa -n kube-system kube-proxy coredns kubernetes-dashboard flannel
kubectl create sa -n kube-system coredns
kubectl create sa -n kube-system kube-proxy
kubectl create sa -n kube-system kubernetes-dashboard
kubectl create sa -n kube-system flannel
```

### Delete pods that need to refresh its service account

```shell
kubectl -n kube-system delete pod -l app=flannel
kubectl -n kube-system delete pod -l k8s-app=kube-proxy
kubectl -n kube-system delete pod -l k8s-app=kube-dns
```

### Drain all nodes

Execute it on master:

```shell
kubectl drain --ignore-daemonsets --delete-local-data $(hostname)
kubectl uncordon $(hostname)
```

And now for each node returned by `kubectl get nodes` execute:

```shell
kubectl drain --ignore-daemonsets --delete-local-data `NODE_NAME`
kubectl uncordon `NODE_NAME`
```

### Validation

Command `kubectl get nodes` should show all nodes in `Ready` state. 
Command `kubectl -n kube-system get pods` should show all system pods in `Running` state.