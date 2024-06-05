# kubernetes-on-rpi cookbook

Inspired by https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Prerequisites

- Get 3 Raspberry PI with at least 4Go RAM
- Install Ubuntu Server 24.04 (noble) using RPI PI Imager

## Methodology

### Get Root Privileges

```bash
sudo su -
```

### Get eeprom configuration

```bash
rpi-eeprom-update
```

### Update packages

```bash
apt update && apt upgrade -y
```

### Install packages

```bash
apt-get -y install socat conntrack ipset linux-raspi
```

### UPDATE CGROUP MEMORY DRIVER in firmware booting files

#### Lire le contenu du fichier /boot/firmware/cmdline.txt à l'aide de la commmande

```bash
cat /boot/firmware/cmdline.txt
```

#### Si nécessaire, ajouter les éléments suivants à la liste existante grâce à la commande sed suivantes

```bash
sed -i '$ s/$/ logo.nologo systemd.unified_cgroup_hierarchy=1 cgroup_no_v1=all cgroup_enable=cpu cgroup_enable=cpuset cgroup_cpu=1 cgroup_cpuset=1 cgroup_enable=memory cgroup_memory=1 loglevel=3 vt.global_cursor_default=0/' /boot/firmware/cmdline.txt
```

#### Rebooter le système

```bash
systemctl reboot
```

### UPDATE IPTABLES CONFIG

- Taper la commande `modprobe overlay && modprobe br_netfilter`

- Pour vérifier que la commande précédente a bien été exécutée, taper les commandes suivantes successivement :

  - `lsmod | grep overlay`
  - `lsmod | grep br_netfilter`

- Configurer iptables afin qu'il voit correctement le traffic ponté
  ```bash
  cat <<EOF | tee /etc/sysctl.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.ipv4.ip_forward                 = 1
  EOF
  ```
- Reload la conf system

  ```bash
  service procps force-reload
  ```

- Vérifier que les paramètres appliqués sont bien pris en compte `sysctl --system`

### Install Containerd

Doc containerd : https://github.com/containerd/containerd/blob/main/docs/getting-started.md

#### Installer containerd

```bash
apt install containerd -y
```

#### Configure containerd to default config

```bash
mkdir -p /etc/containerd/
touch /etc/containerd/config.toml
containerd config default > /etc/containerd/config.toml
```

#### Update default configuration to enable cgroup drive

To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
...
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true

> If the cgroup driver is incorrect, that might lead to the pod in that node always being in a CrashLoopBackOff.

Source : https://stackoverflow.com/questions/75935431/kube-proxy-and-kube-flannel-crashloopbackoff

#### Restard containerd service

```bash
systemctl restart containerd
```

#### Vérification post-install

La CLI Containerd devrait fonctionner pour vérifier taper la commande :

```bash
ctr -v
```

### Install kubeadm, kubectl and kubelet

Source : https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/

```bash
apt-get install -y apt-transport-https ca-certificates curl gpg
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
apt-get update && apt upgrade -y
```

```bash
apt-get install -y kubelet kubeadm kubectl
```

```bash
apt-mark hold kubelet kubeadm kubectl
```

### Run kubeadm to bootstrap cluster (flannel option)

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy --v=5
```

> All pods must be at running status except coredns pods (i.e https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#coredns-is-stuck-in-the-pending-state)

#### Download Flannel YAML configuration file

```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Add following conf to DaemonSet in kube-flannel.yml previously downloaded

```bash
- name: KUBERNETES_SERVICE_HOST
  value: '10.0.0.10'
- name: KUBERNETES_SERVICE_PORT
  value: '6443'
```

#### Install Flannel

Apply flannel conf

```bash
kubectl apply -f kube-flannel.yml
```

#### Activation addon kube-proxy

```bash
kubeadm init phase addon kube-proxy
```

### Add node

```bash
kubeadm join 10.0.0.10:6443 --token s3bz5b.1l7mxw26bp8e2wxt \
        --discovery-token-ca-cert-hash sha256:98e674d6839d5cb6c45ba508e2c9667731bf777f5eeec4a425523e14f1a0bf82 --apiserver-advertise-address=10.0.0.10
```

## Remove node

```bash
kubeadm reset && rm -r $HOME/.kube && iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

## CTR Clean up commands

```bash
ctr -n k8s.io c rm <containers_ids>
ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep etcd) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep sha) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep core) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep kube) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep pause) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep flannel)

```

## Appendix

https://v1-28.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion

### CNI Configuration Tool

https://github.com/containernetworking/cni/tree/main/cnitool

#### Controlling your cluster from machines other than the control-plane node

```bash
kubectl --kubeconfig ./admin.conf get nodes
```

### Proxying API Server to localhost

```bash
kubectl --kubeconfig ./admin.conf proxy
```

| Package name   | Description                                                                          |
| -------------- | ------------------------------------------------------------------------------------ |
| kubeadm        | Installs the /usr/bin/kubeadm CLI tool and the kubelet drop-in file for the kubelet. |
| kubelet        | Installs the /usr/bin/kubelet binary.                                                |
| kubectl        | Installs the /usr/bin/kubectl binary.                                                |
| cri-tools      | Installs the /usr/bin/crictl binary from the cri-tools git repository.               |
| kubernetes-cni | Installs the /opt/cni/bin binaries from the plugins git repository.                  |

### Useful commands

Logs de tous les pods core-dns

```bash
for p in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name);  do kubectl logs --namespace=kube-system $p; done
```

### Troubleshooting

#### CNI TOOL to debug CNI configuration

https://www.cni.dev/docs/cnitool/

#### DNS debugging resolution

https://v1-28.docs.kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

```bash
kubectl get svc --namespace=kube-system
```

```bash
kubectl get endpoints kube-dns --namespace=kube-system -o wide
```

```bash
kubectl get endpointslices.discovery.k8s.io
```

```bash
kubectl get svc -o yaml
```

#### Common repositories to know

| Name                             | Path             |
| -------------------------------- | ---------------- |
| Kubernetes Default Manifest Path | /etc/kubernetes/ |

#### CoreDNS blocked at Running but are not ready

Add Iptables rule to allow connection on 10.0.0.0/8 IP

```bash
iptables -A INPUT -p tcp -m tcp --dport 6443 -s 10.0.0.0/8 -m state --state NEW -j ACCEPT
```

Source : https://forum.linuxfoundation.org/discussion/863356/coredns-pods-are-running-but-not-in-ready-state
