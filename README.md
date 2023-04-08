# K8S on Raspberry PI 4b (4GB)

## Install Packages (Both nodes)

```
sudo apt-get update

sudo apt-get install -y ca-certificates curl apt-transport-https
```

## Install Containerd (Both nodes)

```
wget https://github.com/containerd/containerd/releases/download/v1.7.0/containerd-1.7.0-linux-arm64.tar.gz

sudo tar Cxzvf /usr/local containerd-1.7.0-linux-arm64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service 

sudo mkdir -p /usr/local/lib/systemd/system/

sudo mv containerd.service /usr/local/lib/systemd/system/containerd.service

sudo mkdir -p /etc/containerd/

containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl daemon-reload

sudo systemctl enable --now containerd
```

## Install runc (Both nodes)

```
wget https://github.com/opencontainers/runc/releases/download/v1.1.5/runc.arm64

sudo install -m 755 runc.arm64 /usr/local/sbin/runc
```

## Install CNI plugins (Both nodes)

```
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-arm64-v1.2.0.tgz

sudo mkdir -p /opt/cni/bin 

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.2.0.tgz
```

## Install Kubelet, Kubectl, Kubeadm (Both nodes)

```
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
kubeadm completion bash | sudo tee /etc/bash_completion.d/kubeadm > /dev/null
```


## Persists some required system values (Both nodes)

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply without reboot
sudo sysctl --system
```



## Boostrap Cluster ( Control Plane only)

```
sudo -i
# Define the pod network and the service network as you wish
kubeadm init --pod-network-cidr=10.20.0.0/16  --service-cidr=10.0.0.0/12 --apiserver-cert-extra-sans controller.k8s.local
# Optional if you want to schedule pods on Control nodes
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## CNI - Install calico ( Control Plane only)

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml -O
# fix the CALICO_IPV4POOL_CIDR with the pod network defined when bootstraped the cluster
kubectl apply -f calico.yaml
```

Make sure to configure the config to access the cluster

## Join the nodes (Nodes only)

After bootstrap the cluster, follow the instructions to join the nodes to the cluster.

## Check if the nodes are ready ( Control Plane only)

```
kubectl get nodes -o wide
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
rbp1   Ready    control-plane   31m   v1.26.3   192.168.10.12   <none>        Ubuntu 22.04.2 LTS   5.15.0-1026-raspi   containerd://1.7.0
rbp2   Ready    <none>          30m   v1.26.3   192.168.10.13   <none>        Ubuntu 22.04.2 LTS   5.15.0-1026-raspi   containerd://1.7.0
rbp3   Ready    <none>          30m   v1.26.3   192.168.10.14   <none>        Ubuntu 22.04.2 LTS   5.15.0-1026-raspi   containerd://1.7.0
rbp4   Ready    <none>          30m   v1.26.3   192.168.10.15   <none>        Ubuntu 22.04.2 LTS   5.15.0-1026-raspi   containerd://1.7.0
```

# Extra configuration

This is something that I maybe documet on RBP K8S cluster, but it's working on my old Intel Cluster + Edgerouter SPx

<!-- ### Configure GBP

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
 -->
