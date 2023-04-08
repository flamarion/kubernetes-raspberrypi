## Bootstrap K8S cluster

sudo apt-get update

sudo apt-get install -y ca-certificates curl apt-transport-https

# Containerd

wget https://github.com/containerd/containerd/releases/download/v1.6.14/containerd-1.6.14-linux-amd64.tar.gz

sudo tar Cxzvf /usr/local containerd-1.6.14-linux-amd64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service 

sudo mkdir -p /usr/local/lib/systemd/system/

sudo mv containerd.service /usr/local/lib/systemd/system/containerd.service

sudo mkdir -p /etc/containerd/

containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# sudo sed -i 's/sandbox_image \=.*/sandbox_image = \"registry.k8s.io/pause:3.2\"/g' /etc/containerd/config.toml

sudo systemctl daemon-reload

sudo systemctl enable --now containerd

# runc

wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# CNI plugins

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

sudo mkdir -p /opt/cni/bin 

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

# Kubelet, Kubectl, Kubeadm

## Both nodes
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
kubeadm completion bash | sudo tee /etc/bash_completion.d/kubeadm > /dev/null

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Become root and install the cluster
sudo -i

## Control plane
kubeadm init --pod-network-cidr=10.10.0.0/16  --service-cidr=10.96.0.0/12 --apiserver-cert-extra-sans controller.flama-aws.wandb.ml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

## Follow the instructions for the worker node


## CNI - Install calico

`curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml -O`

fix the `CALICO_IPV4POOL_CIDR` with the `10.10.0.0/16`

`kubectl apply -f calico.yaml`

### Configure GBP

calico-bgp.yaml

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: global
spec:
  peerIP: 192.168.10.1
  asNumber: 1
---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 64512
  serviceLoadBalancerIPs:
  - cidr: 192.168.10.224/27

You can advertise the serviceClusterIPs if you want them reachable directly
```

## Enable metrics

kubectl patch felixconfiguration default --type merge --patch '{"spec":{"prometheusMetricsEnabled": true}}'

kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: felix-metrics-svc
  namespace: kube-system
spec:
  clusterIP: None
  selector:
    k8s-app: calico-node
  ports:
  - port: 9091
    targetPort: 9091
EOF



kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: kube-controllers-metrics-svc
  namespace: kube-system
spec:
  clusterIP: None
  selector:
    k8s-app: calico-kube-controllers
  ports:
  - port: 9094
    targetPort: 9094
EOF


kubectl patch kubecontrollersconfiguration default --type=merge  --patch '{"spec":{"prometheusMetricsPort": 9095}}'


kubectl create -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: calico-monitoring
  labels:
    app:  ns-calico-monitoring
    role: monitoring
EOF

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: calico-prometheus-user
rules:
- apiGroups: [""]
  resources:
  - endpoints
  - services
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-prometheus-user
  namespace: calico-monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-prometheus-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-prometheus-user
subjects:
- kind: ServiceAccount
  name: calico-prometheus-user
  namespace: calico-monitoring
EOF



## LoadBalancer - Install MetalLB



`wget https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml`

Remove all entries related to the `speaker` because the Calico will advertise the BGP an not the speaker


metallb-cm.yaml

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.224/27
```

## Configure Edgerouter

```
set protocols bgp 1 parameters router-id 192.168.10.1
set protocols bgp 1 neighbor 192.168.10.10 remote-as 64512
set protocols bgp 1 neighbor 192.168.10.11 remote-as 64512
commit ; save
exit
show ip route bgp
show ip bgp neighbors
```

## Install NGINX Ingress controller

(Installing the stupid ingress via manifests doesn't work, I still need to investigate it)

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace

kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.103.151.113   192.168.10.225   80:31269/TCP,443:31553/TCP   13h
ingress-nginx-controller-admission   ClusterIP      10.97.241.9      <none>           443/TCP                      13h

### Enable Prometheus metrics

helm upgrade ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --set controller.metrics.enabled=true --set-string controller.podAnnotations."prometheus\.io/scrape"="true" --set-string controller.podAnnotations."prometheus\.io/port"="10254"


## Install Prometheus

