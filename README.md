# kubernetes-multi-master-cluster


## Configure a ha-proxy for load balancer
```
frontend k8s-6443
    bind *:6443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    default_backend k8s_master

backend k8s_master
    balance roundrobin
    mode tcp
    option tcplog
    option tcp-check
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server k8s1 10.10.4.25:6443 check
    server k8s2 10.10.4.10:6443 check
    server k8s3 10.10.4.11:6443 check
```


## Containerd installation:

```
apt-get update
apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
apt-get update  
sudo apt-get -y install containerd 
or
sudo apt-get -y install containerd.io

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

containerd config default >  /etc/containerd/config.toml


## Configuring the systemd cgroup driver
## To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
	
sudo systemctl restart containerd
```

## Deploy Kubernetes Cluster
```
kubeadm init --control-plane-endpoint 10.10.4.29:6443 --upload-certs

kubeadm init --control-plane-endpoint 10.10.4.29:6443 --upload-certs --pod-network-cidr=192.168.0.0/16

Ref: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
```

## Calico Installation:

```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```

