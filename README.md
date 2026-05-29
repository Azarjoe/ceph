# Documentation Projet - Cluster KVM + Stockage CEPH

## Informations générales

- **Projet** : Création laboratoire cluster KVM et stockage CEPH
- **Matière** : M2-t3 - Virtualisation et clustering d'infrastructure et architecture CEPH
- **Enseignant** : Kevin Chevreuil (kchevreuil2@myges.fr)
- **Soutenance** : Samedi 30/05/2026 à 15h45

---

## Architecture

| Node | IP | Rôle |
|---|---|---|
| node1 | 192.168.0.10 | KVM1 + CEPH MON/MGR/MDS/OSD |
| node2 | 192.168.0.20 | KVM2 + CEPH MON/MGR/OSD |
| node3 | 192.168.0.30 | KVM3 + CEPH MON/MGR/MDS/OSD |

---

## Étape 1 - Installation de Rocky Linux 10.1

### Téléchargement de l'ISO
```
https://mirror.karneval.cz/pub/linux/rockylinux/10.1/isos/x86_64/Rocky-10.1-x86_64-minimal.iso
```

### Specs de la VM template dans VMware
- 4 vCPU
- 4 Go de RAM
- 50 Go de stockage
- 1 carte réseau en NAT

### Partitionnement manuel (ext4)
Lors de l'installation Anaconda :
1. Installation Destination → Custom
2. Partition Scheme → Standard Partition
3. Partitions :

| Point de montage | Taille | Système de fichiers |
|---|---|---|
| `/boot` | 1 GiB | ext4 |
| `/boot/efi` | 600 MiB | EFI System Partition |
| `swap` | 4 GiB | swap |
| `/` | ~44 GiB | ext4 |

---

## Étape 2 - Clonage de la VM template

Dans VMware Workstation :
1. Éteindre la VM template
2. Clic droit → Manage → Clone
3. Create a full clone
4. Nommer node1, node2, node3

---

## Étape 3 - Configuration réseau (sur chaque node)

### Définir le hostname
```bash
hostnamectl set-hostname node1  # node2, node3 sur les autres
```

### Identifier l'interface réseau
```bash
ip a  # ens33 dans notre cas
```

### Récupérer la gateway NAT VMware
```bash
ip route | grep default  # 192.168.0.2
```

### Créer le bridge br0
```bash
nmcli connection add type bridge ifname br0 con-name br0
```

### Attribuer une IP statique à br0
```bash
nmcli connection modify br0 \
  ipv4.addresses 192.168.0.10/24 \
  ipv4.gateway 192.168.0.2 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.method manual \
  bridge.stp no
```

### Rattacher ens33 au bridge
```bash
nmcli connection add type bridge-slave \
  ifname ens33 \
  con-name br0-slave \
  master br0
```

### Désactiver l'IP sur ens33
```bash
nmcli connection modify ens33 \
  ipv4.method disabled \
  ipv6.method disabled
```

### Appliquer
```bash
nmcli connection down ens33
nmcli connection up br0-slave
nmcli connection up br0
```

### Ajouter les hosts dans /etc/hosts (sur les 3 nodes)
```bash
cat >> /etc/hosts << 'EOF'
192.168.0.10 node1
192.168.0.20 node2
192.168.0.30 node3
EOF
```

### Plan d'adressage

| VM | IP |
|---|---|
| node1 | 192.168.0.10/24 |
| node2 | 192.168.0.20/24 |
| node3 | 192.168.0.30/24 |

---

## Étape 4 - TPs libvirt

### TP1 - VM avec console série (sur node1)

#### Créer le pool de stockage ISO
```bash
mkdir -p /usr/share/libvirt/iso
virsh pool-define-as iso dir --target /usr/share/libvirt/iso
virsh pool-start iso
virsh pool-autostart iso
```

#### Télécharger l'ISO Debian 13
```bash
wget -P /usr/share/libvirt/iso \
  https://cdimage.debian.org/cdimage/daily-builds/daily/arch-latest/amd64/iso-cd/debian-testing-amd64-netinst.iso
```

#### Créer la VM avec console série
```bash
virt-install \
  --name debian13 \
  --memory 1024 \
  --vcpus 1 \
  --disk path=/var/lib/libvirt/images/debian13.qcow2,format=qcow2 \
  --location /usr/share/libvirt/iso/debian-testing-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics none \
  --serial pty \
  --console pty,target_type=serial \
  --extra-args "console=ttyS0,115200n8" \
  --boot hd,cdrom \
  --noautoconsole
```

#### Se connecter à la console série
```bash
virsh console debian13
```

### TP2 - VM avec VNC (sur node2)

#### Ouvrir le port VNC dans le firewall
```bash
firewall-cmd --add-service=vnc-server --permanent
firewall-cmd --reload
```

#### Créer la VM avec VNC
```bash
virt-install \
  --virt-type kvm \
  --memory 1024 \
  --vcpus 1 \
  --cpu host \
  --cdrom /usr/share/libvirt/iso/debian-testing-amd64-netinst.iso \
  --boot hd,cdrom,menu=on \
  --os-variant debian12 \
  --disk size=20 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
```

Se connecter avec RealVNC sur `192.168.0.20:5900`

### TP3 - Mode rootless (sur node3)

#### Configuration rootless
```bash
# Ajouter dans /etc/environment
echo 'LIBVIRT_DEFAULT_URI=qemu:///session' >> /etc/environment

# Autoriser br0 pour les users non-root
echo "allow br0" > /etc/qemu-kvm/bridge.conf

# Activer le linger pour l'user
loginctl enable-linger <user>
```

#### Vérification
```bash
virsh uri  # doit retourner qemu:///session
```

---

## Étape 5 - Création de l'user projet (sur les 3 nodes)

```bash
useradd -m projet
echo "projet:projet" | chpasswd
```

### Configuration du .bashrc de projet
```bash
echo 'export LIBVIRT_DEFAULT_URI=qemu:///session' >> /home/projet/.bashrc
echo 'export XDG_RUNTIME_DIR=/run/user/$(id -u)' >> /home/projet/.bashrc
loginctl enable-linger projet
```

### Config libvirt pour user projet
```bash
mkdir -p /home/projet/.config/libvirt
echo 'uri_default = "qemu:///session"' > /home/projet/.config/libvirt/libvirt.conf
```

---

## Étape 6 - Ajout des disques CEPH dans VMware

Pour chaque VM dans VMware Workstation :
1. Éteindre la VM
2. Settings → Add → Hard Disk → SCSI
3. Create a new virtual disk → 100 Go → Single file
4. Répéter 3 fois par VM

Vérification après démarrage :
```bash
lsblk  # sdb, sdc, sdd doivent apparaître
```

---

## Étape 7 - Bootstrap du cluster CEPH (sur node1)

### Installer cephadm
```bash
dnf install -y cephadm
```

### Générer la clé SSH pour cephadm
```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

### Bootstrap du cluster
```bash
cephadm bootstrap --mon-ip 192.168.0.10
```

### Récupérer la clé publique cephadm
```bash
ceph cephadm get-pub-key > ~/ceph.pub
```

### Copier la clé sur les 3 nodes
```bash
ssh-copy-id -f -i ~/ceph.pub root@192.168.0.10
ssh-copy-id -f -i ~/ceph.pub root@192.168.0.20
ssh-copy-id -f -i ~/ceph.pub root@192.168.0.30
```

### Configurer le SSH pour cephadm
```bash
cat > ~/.ssh/config << 'EOF'
Host *
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
EOF
ceph cephadm set-ssh-config -i ~/.ssh/config
ceph cephadm set-pub-key -i ~/ceph.pub
```

### Ajouter node2 et node3 au cluster
```bash
ceph orch host add node2 192.168.0.20
ceph orch host add node3 192.168.0.30
```

---

## Étape 8 - Déploiement des MON et MGR

```bash
ceph orch apply mon node1,node2,node3
ceph orch apply mgr node1,node2,node3
```

### Vérification
```bash
ceph status
ceph orch host ls
```

---

## Étape 9 - Ajout des OSDs

```bash
ceph orch apply osd --all-available-devices
```

> **Note** : En production, il faut ajouter les OSDs manuellement :
> ```bash
> ceph orch daemon add osd node1:/dev/sdb
> ceph orch daemon add osd node1:/dev/sdc
> # etc...
> ```

### Vérification
```bash
ceph osd tree
ceph status  # 9 osds: 9 up, 9 in
```

---

## Étape 10 - Création du pool RBD

### Formule PGs
```
Nb OSD x 100 / Nb réplicats = 9 x 100 / 3 = 300
Arrondi à la puissance de 2 supérieure = 512
```

### Créer le pool
```bash
ceph osd pool create rbd 512
ceph osd pool set rbd pg_num 512
ceph osd pool set rbd pgp_num 512
ceph osd pool set rbd pg_autoscale_mode off
ceph osd pool application enable rbd rbd
rbd pool init rbd
```

---

## Étape 11 - Connexion libvirt au pool RBD

### Créer un user CEPH dédié pour libvirt
```bash
ceph auth get-or-create client.libvirt \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd' \
  -o /etc/ceph/ceph.client.libvirt.keyring
chmod 640 /etc/ceph/ceph.client.libvirt.keyring
```

### Récupérer la clé de façon sécurisée
```bash
ceph auth get-key client.libvirt > /tmp/libvirt.key
chmod 600 /tmp/libvirt.key
```

### Créer le secret libvirt
```bash
cat > ~/secret.xml << 'EOF'
<secret ephemeral='no' private='no'>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>
EOF
chmod 600 ~/secret.xml

virsh secret-define ~/secret.xml
virsh secret-set-value <UUID> --file /tmp/libvirt.key
rm /tmp/libvirt.key
```

### Créer le pool libvirt RBD
```bash
cat > ~/pool-rbd.xml << 'EOF'
<pool type="rbd">
  <name>ceph-rbd</name>
  <source>
    <host name="192.168.0.10"/>
    <host name="192.168.0.20"/>
    <host name="192.168.0.30"/>
    <name>rbd</name>
    <auth type="ceph" username="libvirt">
      <secret uuid="a6e8c2b5-b679-4568-bf62-bcd151c0c41b"/>
    </auth>
  </source>
</pool>
EOF
chmod 600 ~/pool-rbd.xml

virsh pool-define ~/pool-rbd.xml
virsh pool-start ceph-rbd
virsh pool-autostart ceph-rbd
```

> Cette configuration est à faire sur les 3 nodes pour l'user `projet`

---

## Étape 12 - Création des VMs sur le pool RBD

```bash
virt-install \
  --virt-type kvm \
  --name debian-rbd \
  --memory 1024 \
  --vcpus 1 \
  --cpu host-model \
  --disk pool=ceph-rbd,size=20 \
  --cdrom /usr/share/libvirt/iso/debian-testing-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics vnc,listen=0.0.0.0,port=5901 \
  --boot hd,cdrom \
  --noautoconsole

virt-install \
  --virt-type kvm \
  --name debian-cephfs \
  --memory 1024 \
  --vcpus 1 \
  --cpu host-model \
  --disk pool=ceph-rbd,size=20 \
  --cdrom /mnt/cephfs/debian-testing-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics vnc,listen=0.0.0.0,port=5900 \
  --boot hd,cdrom \
  --noautoconsole
```

---

## Étape 13 - Live Migration

### Prérequis
- Pool RBD configuré sur les 3 nodes pour user `projet`
- Secret libvirt configuré sur les 3 nodes
- Ports de migration ouverts dans le firewall

```bash
firewall-cmd --add-port=49152-49215/tcp --permanent
firewall-cmd --reload
```

### Clés SSH entre nodes (en user projet)
```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id projet@192.168.0.20
ssh-copy-id projet@192.168.0.30
```

### Aliases dans ~/.bashrc
```bash
alias migrate-rbd1='virsh migrate --live debian-rbd --migrateuri tcp://192.168.0.10 qemu+ssh://projet@192.168.0.10/session'
alias migrate-rbd2='virsh migrate --live debian-rbd --migrateuri tcp://192.168.0.20 qemu+ssh://projet@192.168.0.20/session'
alias migrate-rbd3='virsh migrate --live debian-rbd --migrateuri tcp://192.168.0.30 qemu+ssh://projet@192.168.0.30/session'
alias migrate-cephfs1='virsh migrate --live debian-cephfs --migrateuri tcp://192.168.0.10 qemu+ssh://projet@192.168.0.10/session'
alias migrate-cephfs2='virsh migrate --live debian-cephfs --migrateuri tcp://192.168.0.20 qemu+ssh://projet@192.168.0.20/session'
alias migrate-cephfs3='virsh migrate --live debian-cephfs --migrateuri tcp://192.168.0.30 qemu+ssh://projet@192.168.0.30/session'
```

### Commande de migration
```bash
virsh migrate --live debian-cephfs \
  --migrateuri tcp://192.168.0.20 \
  qemu+ssh://projet@192.168.0.20/session
```

---

## Étape 14 - Création du CephFS

```bash
ceph fs volume create cephfs
ceph fs ls
```

### Vérifier les MDS
```bash
ceph orch ps --daemon-type mds
```

### Configurer les PGs du CephFS
```bash
# Augmenter la limite de PGs par OSD
ceph config set global mon_max_pg_per_osd 1024

# Appliquer 512 PGs sur les pools CephFS
ceph osd pool set cephfs.cephfs.data pg_num 512
ceph osd pool set cephfs.cephfs.data pgp_num 512
ceph osd pool set cephfs.cephfs.meta pg_num 512
ceph osd pool set cephfs.cephfs.meta pgp_num 512
```

---

## Étape 15 - Montage du CephFS

### Créer un user CEPH dédié pour le montage
```bash
ceph auth get-or-create client.mounter \
  mon 'allow r' \
  mds 'allow rw' \
  osd 'allow rw pool=cephfs.cephfs.data, allow rw pool=cephfs.cephfs.meta' \
  -o /etc/ceph/ceph.client.mounter.keyring

ceph auth get-key client.mounter > /etc/ceph/ceph.client.mounter.secret
chmod 600 /etc/ceph/ceph.client.mounter.secret
chmod 640 /etc/ceph/ceph.client.mounter.keyring
```

### Créer le point de montage
```bash
mkdir -p /mnt/cephfs
```

### Copier les fichiers sur node2 et node3
```bash
scp /etc/ceph/ceph.client.mounter.keyring root@192.168.0.20:/etc/ceph/
scp /etc/ceph/ceph.client.mounter.secret root@192.168.0.20:/etc/ceph/
scp /etc/ceph/ceph.client.mounter.keyring root@192.168.0.30:/etc/ceph/
scp /etc/ceph/ceph.client.mounter.secret root@192.168.0.30:/etc/ceph/
```

### Unit systemd pour montage automatique
```bash
cat > /etc/systemd/system/mnt-cephfs.mount << 'EOF'
[Unit]
Description=CephFS mount
After=network.target
AssertPathExists=/etc/ceph/ceph.client.mounter.secret

[Mount]
What=192.168.0.10:/
Where=/mnt/cephfs
Type=ceph
Options=name=mounter,secretfile=/etc/ceph/ceph.client.mounter.secret,_netdev

[Install]
WantedBy=multi-user.target
EOF
```

### Service d'attente du MDS
```bash
cat > /etc/systemd/system/mnt-cephfs-wait.service << 'EOF'
[Unit]
Description=Wait for CEPH MDS before mounting
Before=mnt-cephfs.mount

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'sleep 30 && until ceph mds stat 2>/dev/null | grep -q "up:active"; do sleep 5; done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable mnt-cephfs.mount
systemctl enable mnt-cephfs-wait.service
systemctl start mnt-cephfs-wait.service
systemctl start mnt-cephfs.mount
```

### Copier les units sur node2 et node3
```bash
scp /etc/systemd/system/mnt-cephfs.mount root@192.168.0.20:/etc/systemd/system/
scp /etc/systemd/system/mnt-cephfs.mount root@192.168.0.30:/etc/systemd/system/
scp /etc/systemd/system/mnt-cephfs-wait.service root@192.168.0.20:/etc/systemd/system/
scp /etc/systemd/system/mnt-cephfs-wait.service root@192.168.0.30:/etc/systemd/system/

ssh root@192.168.0.20 "systemctl daemon-reload && systemctl enable mnt-cephfs.mount && systemctl enable mnt-cephfs-wait.service"
ssh root@192.168.0.30 "systemctl daemon-reload && systemctl enable mnt-cephfs.mount && systemctl enable mnt-cephfs-wait.service"
```

---

## Étape 16 - Déplacement des ISOs sur CephFS

```bash
mv /usr/share/libvirt/iso/* /mnt/cephfs/
ls /mnt/cephfs/
```

---

## Étape 17 - Service ACL pour user projet

```bash
cat > /etc/systemd/system/ceph-acl.service << 'EOF'
[Unit]
Description=Set CEPH ACLs for projet user
After=mnt-cephfs.mount network-online.target
Requires=mnt-cephfs.mount
Wants=network-online.target

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 10
ExecStart=/usr/bin/setfacl -m u:projet:r /etc/ceph/ceph.client.admin.keyring
ExecStart=/usr/bin/setfacl -m u:projet:r /etc/ceph/ceph.client.admin.secret
ExecStart=/usr/bin/setfacl -m u:projet:r /etc/ceph/ceph.client.mounter.keyring
ExecStart=/usr/bin/setfacl -m u:projet:r /etc/ceph/ceph.client.mounter.secret
ExecStart=/usr/bin/setfacl -m u:projet:r-x /mnt/cephfs
ExecStart=/usr/bin/setfacl -m u:projet:r /mnt/cephfs/debian-testing-amd64-netinst.iso
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable ceph-acl.service

scp /etc/systemd/system/ceph-acl.service root@192.168.0.20:/etc/systemd/system/
scp /etc/systemd/system/ceph-acl.service root@192.168.0.30:/etc/systemd/system/
ssh root@192.168.0.20 "systemctl daemon-reload && systemctl enable ceph-acl.service"
ssh root@192.168.0.30 "systemctl daemon-reload && systemctl enable ceph-acl.service"
```

---

## Étape 18 - Sécurisation des fichiers

```bash
# Permissions fichiers CEPH
chmod 600 /etc/ceph/ceph.client.admin.keyring
chmod 600 /etc/ceph/ceph.client.admin.secret
chmod 640 /etc/ceph/ceph.client.libvirt.keyring
chmod 640 /etc/ceph/ceph.client.mounter.keyring
chmod 640 /etc/ceph/ceph.client.mounter.secret

# Permissions fichiers home projet
chmod 600 /home/projet/secret.xml
chmod 600 /home/projet/pool-rbd.xml

# Bridge conf
echo "allow br0" > /etc/qemu-kvm/bridge.conf
```

---

## Dashboard CEPH

Accessible sur `https://192.168.0.20:8443/`
Credentials : `admin` / `admin123`

```bash
# Réinitialiser le mot de passe
echo "admin123" | ceph dashboard ac-user-set-password admin -i -
```

---

## Commandes utiles

### État du cluster
```bash
ceph status
ceph osd tree
ceph fs ls
ceph auth list
ceph orch ps
ceph orch host ls
```

### Gestion des VMs
```bash
virsh list --all
virsh start <vm>
virsh destroy <vm>
virsh undefine <vm> --remove-all-storage
```

### Gestion des disques RBD
```bash
rbd ls rbd
rbd rm rbd/<image>
```

### Migration
```bash
# Depuis node1
migrate-cephfs2  # migre vers node2
migrate-cephfs3  # migre vers node3

# Depuis node2
source ~/.bashrc && migrate-cephfs3
```

---

## Récapitulatif sécurité

| Fichier | Permissions | Accessible par |
|---|---|---|
| `ceph.client.admin.keyring` | 600 + ACL | root + projet (lecture) |
| `ceph.client.admin.secret` | 600 + ACL | root + projet (lecture) |
| `ceph.client.libvirt.keyring` | 640 | root uniquement |
| `ceph.client.mounter.keyring` | 640 + ACL | root + projet (lecture) |
| `ceph.client.mounter.secret` | 640 + ACL | root + projet (lecture) |
| `secret.xml` | 600 | projet uniquement |
| `pool-rbd.xml` | 600 | projet uniquement |

---

## Users CEPH

| User | Droits |
|---|---|
| `client.admin` | Super admin CEPH |
| `client.libvirt` | Accès pool `rbd` uniquement |
| `client.mounter` | Accès CephFS uniquement (droits limités) |
