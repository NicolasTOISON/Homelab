# Kubernetes on Raspberry Pi installation book

Inspired by https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Prerequisites

- Get 3 Raspberry PI with at least 4Go RAM
- Install Ubuntu Server 24.04 (noble) using RPI PI Imager (ssh access allowed)

## Knowledge

- Basic linux command (apt, cat, wget, sed, grep and so on...)
- Connect to a host via SSH

## Methodology

### Common manipulations

1. Connect to host by SSH using user credentials
2. Get root privileges

```bash
sudo su -
```

3. Get eeprom configuration (optionnal)

```bash
rpi-eeprom-update
```

4. Update packages

```bash
apt update && apt upgrade -y
```

5. Install packages

```bash
apt-get -y install socat conntrack ipset linux-raspi
```

- socat : [Socat Package Page](https://packages.ubuntu.com/noble/socat)
- conntrack : [Conntrack Package Page](https://packages.ubuntu.com/noble/conntrack)
- ipset : [IPSet Package Page](https://packages.ubuntu.com/noble/ipset)
- linux-raspi : [Socat Package Page](https://packages.ubuntu.com/noble/linux-raspi)

### Prepare host for Kubernetes hosting

1. Update CGROUP MEMORY DRIVER in firmware booting files

- Read file's content located at /boot/firmware/cmdline.txt using following command :

  ```bash
  cat /boot/firmware/cmdline.txt
  ```

2. (If necessary), add following parameters at the end of the file using the following command :

```bash
sed -i '$ s/$/ logo.nologo systemd.unified_cgroup_hierarchy=1 cgroup_no_v1=all cgroup_enable=cpu cgroup_enable=cpuset cgroup_cpu=1 cgroup_cpuset=1 cgroup_enable=memory cgroup_memory=1 loglevel=3 vt.global_cursor_default=0/' /boot/firmware/cmdline.txt
```

3. Reboot system to applyu change on firmware booting file

```bash
systemctl reboot
```

4. Update iptables config by typing following command :

```bash
modprobe overlay && modprobe br_netfilter
```

5. Verify that the previous command has been correctly executed by using theses commands :

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

6. Add custom iptables configuration

```bash
cat <<EOF | tee /etc/sysctl.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
```

7. Reload system configuration

```bash
service procps force-reload
```

8. Verify that `net.bridge.bridge-nf-call-iptables` and `net.ipv4.ip_forward` are correctly applied by checking system configuration :

```bash
sysctl --system
```

#### Install Containerd

<cite>Doc containerd : https://github.com/containerd/containerd/blob/main/docs/getting-started.md<cite>

1. Install containerd and runc using apt package manager

```bash
apt install containerd -y
```

2. Configure containerd to default configuration

```bash
mkdir -p /etc/containerd/
touch /etc/containerd/config.toml
containerd config default > /etc/containerd/config.toml
```

3. Update default configuration to enable cgroup drive

To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
...
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true # <-- Line to modify
```

> **N.B :**
>
> If the cgroup driver is incorrect, that might lead to the pod in that node always being in a CrashLoopBackOff.
>
> <cite>Source : https://stackoverflow.com/questions/75935431/kube-proxy-and-kube-flannel-crashloopbackoff</cite>

4. Restart containerd service

```bash
systemctl restart containerd
```

5. Post-install verification

```bash
ctr -v
```

#### Install kubeadm, kubectl and kubelet

<cite>Source : https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/<cite>

1. Install dependencies packages of kubeadm, kubectl and kubelet.

```bash
apt-get install -y apt-transport-https ca-certificates curl gpg
```

- [apt-transport-https package documentation](https://packages.ubuntu.com/noble/apt-transport-https)
- [ca-certificates package documentation](https://packages.ubuntu.com/noble/ca-certificates)
- [curl package documentation](https://packages.ubuntu.com/noble/curl)
- [gpg package documentation](https://packages.ubuntu.com/noble/gpg)

2. Download the public signing key for the k8s repository

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

3. Add k8s repository to apt sources repositories

```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update and ugprade package list

```bash
apt-get update && apt upgrade -y
```

5. Install kubelet, kubeadm and kubectl packages

```bash
apt-get install -y kubelet kubeadm kubectl
```

6. Hold kubelet, kubeadm and kubectl version to the current one to avoid major version upgrade and cause cluster crash

```bash
apt-mark hold kubelet kubeadm kubectl
```

| Package name   | Description                                                                          |
| -------------- | ------------------------------------------------------------------------------------ |
| kubeadm        | Installs the /usr/bin/kubeadm CLI tool and the kubelet drop-in file for the kubelet. |
| kubelet        | Installs the /usr/bin/kubelet binary.                                                |
| kubectl        | Installs the /usr/bin/kubectl binary.                                                |
| cri-tools      | Installs the /usr/bin/crictl binary from the cri-tools git repository.               |
| kubernetes-cni | Installs the /opt/cni/bin binaries from the plugins git repository.                  |

### Bootstrap cluster control plane (flannel option)

1. Init K8S Control Plane

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy --v=5
```

> All pods must be at running status except coredns pods
>
> Source : <cite>[K8S Troubleshooting Documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#coredns-is-stuck-in-the-pending-state)</cite>

2. Download Flannel YAML configuration file

```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

3. Update flannel configuration by adding following conf to DaemonSet in kube-flannel.yml :

```bash
- name: KUBERNETES_SERVICE_HOST
  value: '10.0.0.10'
- name: KUBERNETES_SERVICE_PORT
  value: '6443'
```

4. Install Flannel CNI

```bash
kubectl apply -f kube-flannel.yml
```

5. Activation addon kube-proxy

```bash
kubeadm init phase addon kube-proxy
```

### Add worker nodes

1. Run the following command :

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

- To get token :

```bash
kubeadm token list
```

or if the token has expired

```bash
kubeadm token create
```

- To get hash :

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

## Clean up

### Remove node

```bash
kubeadm reset && rm -r $HOME/.kube && iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### CTR containers clean up

```bash
ctr -n k8s.io c rm <containers_ids>
ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep etcd) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep sha) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep core) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep kube) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep pause) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep flannel)

```

## Appendix

<cite>[Kubernetes Documentation on Shell Autocompletion](https://v1-28.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion)</cite>

### CNI Configuration Tool

[CNI Tool Github Repo](https://github.com/containernetworking/cni/tree/main/cnitool)
[CNI Tool Website](https://www.cni.dev/docs/cnitool/)

### Controlling your cluster from machines other than the control-plane node

```bash
kubectl --kubeconfig ./admin.conf get nodes
```

### Proxying API Server to localhost

```bash
kubectl --kubeconfig ./admin.conf proxy
```

## Useful commands

Logs de tous les pods core-dns

```bash
for p in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name);  do kubectl logs --namespace=kube-system $p; done
```

## Common repositories to know

| Name                             | Path             |
| -------------------------------- | ---------------- |
| Kubernetes Default Manifest Path | /etc/kubernetes/ |

## Troubleshooting

#### DNS debugging resolution

<cite>https://v1-28.docs.kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/</cite>

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

#### CoreDNS blocked at Running but are not ready

Add Iptables rule to allow connection on 10.0.0.0/8 IP

```bash
iptables -A INPUT -p tcp -m tcp --dport 6443 -s 10.0.0.0/8 -m state --state NEW -j ACCEPT
```

Source : https://forum.linuxfoundation.org/discussion/863356/coredns-pods-are-running-but-not-in-ready-state
