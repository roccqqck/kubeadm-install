# Containerd runtime
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

# install containerd runc cni-plugin
https://github.com/containerd/containerd/blob/main/docs/getting-started.md


https://docs.docker.com/engine/install/ubuntu/
The ```containerd.io``` packages in DEB and RPM formats are distributed by Docker (not by the containerd project). See the Docker documentation for how to set up apt-get or dnf to install ```containerd.io``` packages:
The containerd.io package contains ```runc``` too, but does not contain ```CNI plugins```.



```
sudo apt-get install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt install containerd.io -y
```


# systemd containerd.service
```
systemctl status containerd
```
If you intend to start containerd via systemd, you should also download the ```containerd.service``` unit file from https://github.com/containerd/containerd/blob/main/containerd.service into ```/usr/local/lib/systemd/system/containerd.service```, and run the following commands:

```
sudo vim /lib/systemd/system/containerd.service
```
```
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```


```
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl restart containerd
systemctl status containerd
```


### Installing CNI plugins
Download the cni-plugins-<OS>-<ARCH>-<VERSION>.tgz archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under ```/opt/cni/bin```:
```
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```
```
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```




## Customizing containerd
containerd uses a configuration file located in ```/etc/containerd/config.toml``` for specifying daemon level options. A sample configuration file can be found here.
https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md

The default configuration can be generated via ```containerd config default > /etc/containerd/config.toml```.

```
containerd config default > /etc/containerd/config.toml
```





### Configuring the systemd cgroup driver
To use the systemd cgroup driver in ```/etc/containerd/config.toml``` with runc, set

```
sudo vim /etc/containerd/config.toml
```
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
If you apply this change, make sure to restart containerd:
```
sudo systemctl restart containerd
```