# kubernetes-on-rpi cookbook

## PREPARE RPI HOSTS

### Install RPI OS using RPI PI Imager

### Get Root Privileges
`sudo su`

### READ EEPROM CONFIGURATION
`rpi-eeprom-update`

### UPDATE EEPROM (optional)

- Launch RPI Software configuration tool `sudo raspi-config`
- Selectionner l'option 6 nommé Advanced Options puis l'option A5 nommé bootloader version puis sélectionner l'option E1 - latest.
- Rebooté le système `sudo systemctl reboot`

### Update Packages
`apt update && apt upgrade -y && rpi-update`

### DISABLE SWAP MEMORY (if necessary)

- Afin de vérifier que la mémoire swap est utilisé taper la commande :
  `swapon --show`
- Si la sortie de la commande précédente passer à l'étape suivante sinon il faut modifier le paramétrage. Pour cela exécuter les commandes suivantes :
  - `dphys-swapfile swapoff`
  - `dphys-swapfile uninstall`
  - `update-rc.d dphys-swapfile remove`
  - `apt purge dphys-swapfile -y && apt autoremove`
  - `sysctl -w vm.swappiness=0`

### UPDATE CGROUP MEMORY DRIVER in firmware booting files

- Lire le contenu du fichier /boot/firmware/cmdline.txt à l'aide de la commmande `cat /boot/firmware/cmdline.txt`
- Si nécessaire, ajouter les éléments suivants à la liste existante :`cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1` grâce à la commande sed suivantes
- `sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt`
- Rebooter le système `systemctl reboot`

### UPDATE IPTABLES CONFIG

- Rajouter overlay et br_netfilter à la conf k8s :
  ```bash
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF
  ```
- taper la commande `modprobe overlay &&modprobe br_netfilter`
- Pour vérifier que la commande précédente a bien été exécutée, taper les commandes suivantes successivement :

  - `lsmod | grep overlay`
  - `lsmod | grep br_netfilter`

- Configurer iptables afin qu'il voit correctement le traffic ponté

  ```bash
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  ```

- Vérifier que les paramètres appliqués sont bien pris en compte `sysctl --system`

### INSTALL CRI

Doc containerd : https://github.com/containerd/containerd/blob/main/docs/getting-started.md

#### Installer containerd

Dépôt des release contaienrd : https://containerd.io/downloads/

- Télécharger containerd
  `wget https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-arm64.tar.gz`
- Dézipper l'archive dans /usr/local : `sudo tar Cxzvf /usr/local containerd-1.7.14-linux-arm64.tar.gz`
- Téleçharger l'unit systèmed de containerd : `wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service`
- Déplacer le fichier précédemment télécharger vers la destination /usr/local/lib/systemd/system : `sudo mv containerd.service /usr/local/lib/systemd/system/`
- Vérification que la copie a bien eu lieu : `sudo ls -al /usr/local/lib/systemd/system | grep containerd`
- Configurer containerd par défaut à l'aide de la commande suivante : `sudo containerd config default > /etc/containerd/config.toml`
- Recharger le daemon systemd : `sudo systemctl daemon-reload`
- Activer l'unit systemd de containerd : `sudo systemctl enable --now containerd`

#### Installer runc

Dépôt des releases runc : https://github.com/opencontainers/runc/releases

- Télécharger runc : `wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.arm64`
- Installer runc : `sudo install -m 755 runc.arm64 /usr/local/sbin/runc`

#### Installer le plugin CNI

Dépôt des release de CNI Plugins : https://github.com/containernetworking/plugins/releases

bin_dir = "/opt/cni/bin"

conf_dir = "/etc/cni/net.d"

- Téleçharger le plugin CNI : `wget https://github.com/containernetworking/plugins/releases/download/v1.4.1/cni-plugins-linux-arm64-v1.4.1.tgz`
- Créer le répertoire /cni/bin/ : `sudo mkdir -p /opt/cni/bin`
- Dézipper le fichier dans le répertoire précédemment créé : `sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.4.1.tgz`
- Vérifier la conf CNI : `sudo ls -al /etc/cni/net.d/` - Si pas de fichier de conf exécuter la commande :

  ```bash
  cat << EOF | tee /etc/cni/net.d/10-k8s-custom-network.conf
  {
  "cniVersion": "1.0.0",
  "name": "k8s-custom-network",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "promiscMode": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{
            "subnet": "10.244.0.0/16"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
  }
  EOF
  ```
  
*Inspired by : https://github.com/containerd/containerd/blob/main/script/setup/install-cni & https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/#an-example-containerd-configuration-file*

  ~~puis rajouter le fichier de configuration de l'interface loopback nécessaire :~~

  ```bash
  cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
  {
    "cniVersion": "1.0.0",
    "name": "lo",
    "type": "loopback"
  }
  EOF
  ```


#### Vérification post-install

La CLI Containerd devrait fonctionner pour vérifier taper la commande : `ctr -v`

### INSTALL KUBEADM, KUBECTL and KUBELET

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt upgrade -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
````

### Run kubeadm to bootstrap cluster (cilium option)
``` bash
kubeadm init --pod-network-cidr=10.1.1.0/24 --skip-phases=addon/kube-proxy
```
All pods must be at running status except coredns pods (i.e https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#coredns-is-stuck-in-the-pending-state)

#### Download Cilium binaries
``` bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

#### Install Cilium binaries
``` bash
cilium install --version 1.15.2
```

### Run kubeadm to bootstrap cluster (flannel option)
``` bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy
```
All pods must be at running status except coredns pods (i.e https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#coredns-is-stuck-in-the-pending-state)

#### Deownload Flannel YAML configuration file
```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Add following conf to DaemonSet in kube-flannel.yml previously downloaded
``` bash
- name: KUBERNETES_SERVICE_HOST
  value: '<IP Master/DNS Master>' #ip address or dns of the host where kube-apiservice is running
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

### To check further
https://docs.cilium.io/en/stable/operations/system_requirements/

https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/

### Appendix

https://v1-28.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion

#### CNI Configuration Tool
https://github.com/containernetworking/cni/tree/main/cnitool
