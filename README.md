# kubernetes-on-rpi cookbook

## PREPARE RPI HOSTS

### Install RPI OS using RPI PI Imager

### Get Root Privileges
`sudo su`

### READ EEPROM CONFIGURATION
`rpi-eeprom-update`

### Update Packages
`apt update && apt upgrade -y`

### UPDATE CGROUP MEMORY DRIVER in firmware booting files
- Lire le contenu du fichier /boot/firmware/cmdline.txt à l'aide de la commmande `cat /boot/firmware/cmdline.txt`
- Si nécessaire, ajouter les éléments suivants à la liste existante :`cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1` grâce à la commande sed suivantes
- `sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt`
- Rebooter le système `systemctl reboot`

### UPDATE IPTABLES CONFIG
- Rajouter overlay et br_netfilter à la conf k8s :
  ```bash
  cat <<EOF | tee /etc/modules-load.d/local.conf
  overlay
  br_netfilter
  EOF
  ```
- Taper la commande `modprobe overlay && modprobe br_netfilter`
- Pour vérifier que la commande précédente a bien été exécutée, taper les commandes suivantes successivement :
  - `lsmod | grep overlay`
  - `lsmod | grep br_netfilter`

- Configurer iptables afin qu'il voit correctement le traffic ponté
  ```bash
  cat <<EOF | tee /etc/sysctl.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  ```
  
- Reload la conf system
  ```bash
  service procps force-reload
  ```

- Vérifier que les paramètres appliqués sont bien pris en compte `sysctl --system`

### INSTALL CRI

Doc containerd : https://github.com/containerd/containerd/blob/main/docs/getting-started.md

#### Installer containerd

Dépôt des release contaienrd : https://containerd.io/downloads/

- Télécharger containerd
  `wget https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-arm64.tar.gz`
- Dézipper l'archive dans /usr/local : `tar Cxzvf /usr/local containerd-1.7.14-linux-arm64.tar.gz`
- Téleçharger l'unit systèmed de containerd : `wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service`
- Créer le répertoire /systemd/system/
  ```bash
  mkdir -p /usr/local/lib/systemd/system/
  ```
- Déplacer le fichier précédemment télécharger vers la destination /usr/local/lib/systemd/system : `mv containerd.service /usr/local/lib/systemd/system/`
- Vérification que la copie a bien eu lieu : `ls -al /usr/local/lib/systemd/system | grep containerd`
- Créer le répertoire /etc/containerd
  ```bash
  mkdir -p /etc/containerd/
  ```
- Configurer containerd par défaut à l'aide de la commande suivante : `containerd config default > /etc/containerd/config.toml`
- Recharger le daemon systemd : `systemctl daemon-reload`
- Activer l'unit systemd de containerd : `systemctl enable --now containerd`

#### Installer runc

Dépôt des releases runc : https://github.com/opencontainers/runc/releases

- Télécharger runc : `wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.arm64`
- Installer runc : `install -m 755 runc.arm64 /usr/local/sbin/runc`

#### Vérification post-install

La CLI Containerd devrait fonctionner pour vérifier taper la commande : `ctr -v`

### INSTALL KUBEADM, KUBECTL and KUBELET

```bash
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt upgrade -y
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
````

### Run kubeadm to bootstrap cluster (cilium option)
``` bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy --apiserver-advertise-address=10.0.0.10
```
All pods must be at running status except coredns pods (i.e https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#coredns-is-stuck-in-the-pending-state)

#### Deownload Flannel YAML configuration file
```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Add following conf to DaemonSet in kube-flannel.yml previously downloaded
``` bash
- name: KUBERNETES_SERVICE_HOST
  value: '10.0.0.10'
- name: KUBERNETES_SERVICE_PORT
  value: '6443'
```
#### Install Flannel
Apply flannel conf
``` bash
kubectl apply -f kube-flannel.yml
```

### Remove node
```bash
kubeadm reset && rm -r $HOME/.kube && iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### CTR Clean up commands
```bash
ctr -n k8s.io c rm <containers_ids>
ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep etcd) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep sha) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep core) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep kube) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep pause)
```

### Appendix

https://v1-28.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion

#### CNI Configuration Tool
https://github.com/containernetworking/cni/tree/main/cnitool
