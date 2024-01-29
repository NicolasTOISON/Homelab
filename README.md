# kubernetes-on-rpi cookbook

## PREPARE RPI HOSTS

### Install RPI OS using RPI PI Imager

### Update Packages

`sudo apt update && sudo apt upgrade -y`

### READ EEPROM CONFIGURATION

`sudo rpi-eeprom-update`

### UPDATE EEPROM

- Launch RPI Software configuration tool `sudo raspi-config`
- Selectionner l'option 6 nommé Advanced Options puis l'option A5 nommé bootloader version puis sélectionner l'option E1 - latest.
- Rebooté le système `sudo systemctl reboot`

### DISABLE SWAP MEMORY

- Afin de vérifier que la mémoire swap est utilisé taper la commande :
  `sudo swapon --show`
- Si la sortie de la commande précédente passer à l'étape suivante sinon il faut modifier le paramétrage. Pour cela exécuter les commandes suivantes :
  - `sudo dphys-swapfile swapoff`
  - `sudo dphys-swapfile uninstall`
  - `sudo update-rc.d dphys-swapfile remove`
  - `sudo apt purge dphys-swapfile -y && sudo apt autoremove`
  - `sudo sysctl -w vm.swappiness=0`

### UPDATE CGROUP MEMORY DRIVER in firmware booting files

- Lire le contenu du fichier /boot/firmware/cmdline.txt à l'aide de la commmande `sudo vi /boot/firmware/cmdline.txt`
- Si nécessaire, ajouter les éléments suivants à la liste existante :`cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`
- Rebooter le système `sudo systemctl reboot`

### UPDATE IPTABLES CONFIG

- Rajouter overlay et br_netfilter à la conf k8s :
  ```bash
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF
  ```
- taper la commande `sudo modprobe overlay && sudo modprobe br_netfilter`
- Pour vérifier que la commande précédente a bien été exécutée, taper les commandes suivantes successivement :

  - `lsmod | grep overlay`
  - `lsmod | grep br_netfilter`

- Configurer iptables afin qu'il voit correctement le traffic ponté

  ```bash
  sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  ```

- Vérifier que les paramètres appliqués sont bien pris en compte `sudo sysctl --system`

### INSTALL CRI

Doc containerd : https://github.com/containerd/containerd/blob/main/docs/getting-started.md

#### Installer containerd

Dépôt des release contaienrd : https://containerd.io/downloads/

- Télécharger containerd
  `wget https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-arm64.tar.gz`
- Dézipper l'archive dans /usr/local : `sudo tar Cxzvf /usr/local containerd-1.7.11-linux-arm64.tar.gz`
- Téleçharger l'unit systèmed de containerd : `wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service`
- Déplacer le fichier précédemment télécharger vers la destination /usr/local/lib/systemd/system : `sudo mv containerd.service /usr/local/lib/systemd/system/`
- Vérification que la copie a bien eu lieu : `sudo ls -al /usr/local/lib/systemd/system | grep containerd`
- Configurer containerd par défaut à l'aide de la commande suivante : `sudo containerd config default > /etc/containerd/config.toml`
- Recharger le daemon systemd : `sudo systemctl daemon-reload`
- Activer l'unit systemd de containerd : `sudo systemctl enable --now containerd`

#### Installer runc

Dépôt des releases runc : https://github.com/opencontainers/runc/releases

- Télécharger runc : `wget https://github.com/opencontainers/runc/releases/download/v1.1.11/runc.arm64`
- Installer runc : `sudo install -m 755 runc.arm64 /usr/local/sbin/runc`

#### Installer le plugin CNI

Dépôt des release de CNI Plugins : https://github.com/containernetworking/plugins/releases

bin_dir = "/opt/cni/bin"

conf_dir = "/etc/cni/net.d"

- Téleçharger le plugin CNI : `wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-arm64-v1.4.0.tgz`
- Créer le répertoire /cni/bin/ : `sudo mkdir -p /opt/cni/bin`
- Dézipper le fichier dans le répertoire précédemment créé : `sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.4.0.tgz`
- Vérifier la conf CNI : `sudo ls -al /etc/cni/net.d/` - Si pas de fichier de conf exécuter la commande :

  ```bash
  cat << EOF | tee /etc/cni/net.d/10-containerd-net.conflist
  {
  "cniVersion": "1.0.0",
  "name": "containerd-net",
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
            "subnet": "10.88.0.0/16"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" },
          { "dst": "::/0" }
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

  puis rajouter le fichier de configuration de l'interface loopback nécessaire :

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
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update && sudo apt upgrade -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
````

### Remove node
```bash
kubeadm reset && rm -r $HOME/.kube && iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### CTR Clean up commands
```bash
ctr -n k8s.io c rm <containers_ids>
ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep etcd) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep sha) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep core) ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep kube) && ctr -n k8s.io i rm $(ctr -n k8s.io i ls -q | grep pause)
```
