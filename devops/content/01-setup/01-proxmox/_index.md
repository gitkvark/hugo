+++
categories = ['proxmox']
description = 'Installation et configuration du serveur Proxmox'
title = 'Proxmox'
weight = 1
+++

### Installation Proxmox

Vous pouvez utiliser une machine ancienne qui ne sert plus à rien, mais si vous n'en avez pas, il est possible de louer un serveur chez OVH. Assurez-vous que la machine dispose d'au moins 32 Go de RAM et d'un SSD pour des performances optimales.

Une fois le serveur prêt sur OVH, installez Proxmox à partir du site web de gestion : www.ovh.com/manager. La version 8 est disponible, et vous pouvez configurer la partition (RAID, etc.) avant l'installation.

J'ai configuré une interface réseau de type `bridge` nommée `vmbr1` pour relier les machines virtuelles éventuelles. J'ai utilisé un CIDR assez large (/16) afin d'avoir suffisamment d’adresses IP disponibles.

![Configuration du réseau](01-proxmox-network-page.png?width=60pc)

J'ai créé une machine virtuelle avec Debian, que j'ai nommée `bastion`. Cette machine me permet de me connecter aux machines virtuelles bientôt déployées pour constituer le cluster Kubernetes.

### Préparation du serveur Proxmox pour l'automatisation

Pour que OpenTofu (ou Terraform) puisse gérer le serveur Proxmox, il faut créer deux comptes. L'API de Proxmox 8 étant limitée, le provider `bpg/proxmox` utilise à la fois l'API et une connexion SSH avec `sudo` pour les tâches non disponibles via l'API. On va donc créer un compte PVE et un compte PAM, puis configurer les privilèges au sein de Proxmox ainsi que sur l'interface SSH.

#### Configuration de l'utilisateur PVE

On se connecte à la machine Proxmox via le CLI en tant qu'utilisateur "root".

D'abord, on crée le rôle :

```bash
pveum role add TerraformRole -privs "Datastore.Allocate Datastore.AllocateSpace Datastore.Audit Datastore.AllocateTemplate Pool.Allocate Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.Migrate VM.Monitor VM.PowerMgmt SDN.Use"
```

Puis on crée l'utilisateur PVE, créé un mot de passe et attribue le rôle :

```bash
pveum user add terraform@pve -comment "Terraform user"
pveum passwd terraform@pve
pveum aclmod / -user terraform@pve -role TerraformRole
```

Nous aurons besoin d'un jeton :

```bash
pveum user token add terraform@pve terraform -expire 0 -privsep 0 -comment "Terraform token"
```

Il est important de sauvegarder le jeton, car il ne sera affiché qu'une seule fois.

#### Configuration de l'utilisateur PAM

Proxmox nécessite une connexion SSH pour les commandes qui ne sont pas encore disponibles via l’API.

On reste connecté à la machine Proxmox en tant que "root". On crée l'utilisateur :

```bash
useradd -m terraform
```

On doit autoriser certaines commandes qui sont nécessaires pour les tâches :

```bash
visudo -f /etc/sudoers.d/terraform
```

On copie les privilèges dans l'éditeur :

```bash
terraform ALL=(root) NOPASSWD: /sbin/pvesm
terraform ALL=(root) NOPASSWD: /sbin/qm
terraform ALL=(root) NOPASSWD: /usr/bin/tee /var/lib/vz/*
```

Nous aurons besoin d'une clé SSH que OpenTofu utilisera pour se connecter sans mot se passe. On copie la clé publique dans le fichier `~/.ssh/authorized_keys` sous l'utilisateur récemment créé (par exemple `terraform`).

On peut alors tester la connexion depuis (par exemple) la machine virtuelle `bastion` :

```bash
ssh terraform@<proxmox-serveur> sudo pvesm apiinfo 
```

Si tout va bien, on aura la réponse suivante :

```bash
devopsdemystifie@bastion:~/src$ ssh terraform@10.0.0.1 sudo pvesm apiinfo
APIVER 10
APIAGE 1
```

> [!NOTE]
> La clé privée doit être importée dans la mémoire :
> ```bash
> eval $(ssh-agent)
> ssh-add ~/.ssh/id_ed25519
> ```
