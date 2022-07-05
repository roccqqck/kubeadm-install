```
vagrant status
```
```
vagrant up
```

```
vagrant ssh kubemaster
vagrant ssh kubenode01
vagrant ssh kubenode02
```

# 在安裝 k8s 前，必須把所有 node 的 swap disable 。
```
sudo swapoff -a
sudo vim /etc/fstab
```
```
# /swapfile ... ...
```



```
sudo apt update
sudo apt upgrade
```



# Letting iptables see bridged traffic
```
lsmod | grep br_netfilter
sudo modprobe br_netfilter
```

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```


# install docker-ce
https://docs.docker.com/engine/install/ubuntu/
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker vagrant
newgrp docker 
```



# 更新docker daemon.json

```
sudo vim /etc/docker/daemon.json
```
```
{
    "exec-opts":["native.cgroupdriver=systemd"],
    "log-driver":"json-file",
    "log-opts":{
        "max-size":"100m"
    },
    "storage-driver":"overlay2"
}
```
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```



# install kubeadm

Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Download the Google Cloud public signing key:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
Add the Kubernetes apt repository:
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet=1.23.6-00 kubeadm=1.23.6-00 kubectl=1.23.6-00
sudo apt-mark hold kubelet kubeadm kubectl
```





# kueadm init
```
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
```


```
kubeadm join 192.168.56.2:6443 --token 6h7pc2.zltywbesxrcu5mel \
	--discovery-token-ca-cert-hash sha256:90de65e66603664f390a83170b7135e4ed710bf248e48d87484227aa266de6c1
```

create token
```
kubeadm token create
```
show hash
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

To start using your cluster, you need to run the following as a regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
kubectl get nodes -o wide
```


# kubelet log
```
journalctl -f -u kubelets
```

## if there is something wrong with ```kubeadm join```
```
systemctl stop ufw
systemctl disable ufw
reboot
```
masternodes ping workernodes

workernodes ping masternodes

workernodes curl masternodes:6443
```
curl -vk https://192.168.56.2:6443
```


# Install CNI
## Calico (network policy)
https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart

https://docs.projectcalico.org/v3.8/getting-started/kubernetes/installation/calico

https://docs.projectcalico.org/v3.8/manifests/calico.yaml

https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml

需手動更改pod cidr網段  192.168.0.0/16 -> 10.244.0.0/16

Install Calico
Install the Tigera Calico operator and custom resource definitions.
```
curl -OL https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```
Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest, see the installation reference.

```
curl -OL https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```
```
POD_CIDR="10.244.0.0/16" \
sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" custom-resources.yaml
```
```
kubectl apply -f tigera-operator.yaml
kubectl apply -f custom-resources.yaml
```
Note: Before creating this manifest, read its contents and make sure its settings are correct for your environment. For example, you may need to change the default IP pool CIDR to match your pod network CIDR.







## fannel (without network policy) 輕量
https://github.com/flannel-io/flannel#flannel

https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

## weave 更輕量
https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```




```
kubectl get pods -A
```

## test
```
kubectl run nginx --image=nginx:stable
kubectl create deploy nginx --image=nginx:stable
```