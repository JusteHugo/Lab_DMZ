# Lab Architecture Réseau & DMZ

Ce projet documente la mise en place d'une infrastructure réseau segmentée comprenant un pare-feu sous Linux, une zone démilitarisée (DMZ) et un réseau local (LAN).

## Informations de connexion (Notes de Lab)
*   **Firewall :** `fw` / `#JusteHugoFW`
*   **DMZ :** `vboxuser` / `#JusteHugoDMZ`
*   **LAN :** `student` (Machine Labtainer, à changer quand j'aurais plus de stockage)

---

## Étape 1 : Configuration des interfaces réseau (Hyperviseur)

*   **Firewall (3 interfaces) :**
    *   Carte 1 : NAT (Joue le rôle du lien WAN vers Internet)
    *   Carte 2 : Réseau Interne -> `net-dmz`
    *   Carte 3 : Réseau Interne -> `net-lan`
*   **Serveur DMZ (1 interface) :** Réseau Interne -> `net-dmz`
*   **Client LAN (1 interface) :** Réseau Interne -> `net-lan`

> **Note sur les hyperviseurs :** Le mode "Réseau Interne" est spécifique à VirtualBox. Sur VMware, il faut utiliser l'option "LAN Segment". Sur un hyperviseur de type 1 comme Proxmox VE, il faut créer des ponts virtuels distincts (Linux Bridge) sans port physique associé.

---

## Étape 2 : Configuration de l'adressage IP

### 2.1 - Fichiers de configuration réseau (YAML & Interfaces)

**Serveur DMZ** 
Fichier : `sudo nano /etc/netplan/00-installer-config.yaml`
```yaml
network:
  ethernets:
    enp0s3:
      match:
        macaddress: 08:00:27:2e:ec:a4
      set-name: enp0s3
      dhcp4: false
      addresses:
        - 10.10.10.10/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

**Firewall**
Fichier : `sudo nano /etc/netplan/00-installer-config.yaml`
```yaml
network:
  ethernets:
    enp0s3:
      match:
        macaddress: [MAC_WAN_A_COMPLETER]
      set-name: enp0s3
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 10.10.10.254/24
    enp0s9:
      dhcp4: false
      addresses:
        - 192.168.100.254/24
  version: 2
```

**LAN**
Fichier : `sudo nano /etc/netplan/00-installer-config.yaml`
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.100.10/24
      routes:
        - to: default
          via: 192.168.100.254
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  version: 2
```

**Client LAN (Environnement sans Netplan)**
Fichier : `sudo nano /etc/network/interfaces`
```text
auto enp0s3
iface enp0s3 inet static
    address 192.168.100.10
    netmask 255.255.255.0
    gateway 192.168.100.254
    dns-nameservers 8.8.8.8 1.1.1.1
```


### 2.2 - Commandes d'application (Update des statuts)

**Sur la DMZ et le Firewall (Application Netplan) :**
```bash
sudo netplan apply
```

**Sur la machine LAN (Nettoyage et attribution manuelle) :**
```bash
sudo ip addr flush dev enp0s3
sudo ip link set enp0s3 up
sudo ip addr add 192.168.100.10/24 dev enp0s3
sudo ip route add default via 192.168.100.254
```

---

## Étape 3 : Activation du routage (IP Forwarding)

Par défaut, le noyau Linux refuse de faire transiter des paquets d'une carte réseau à une autre (sécurité native) [Je crois j'ai vu passer ça sur reddit et son code à corriger mon problème :) ]. Pour transformer notre machine Firewall en routeur, il faut activer l'IP Forwarding.

**Modification directe dans le noyau :**
```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

**Vérification :**
```bash
cat /proc/sys/net/ipv4/ip_forward
```
*(La commande doit retourner `1` pour confirmer le routage).*

---

## Étape 4 : Politique Zero Trust avec nftables

Une fois le routage actif, le LAN peut communiquer avec la DMZ, ce qui représente un risque de sécurité critique. L'objectif est de mettre en place un pare-feu `nftables` (je n'ai pas assez de RAM pour PfSense et les dizaines de pages  braves de docs) pour bloquer les flux initiés par la DMZ.

**1. Édition de la "matrice de flux" :** 
Fichier : `sudo nano /etc/nftables.conf`

```text
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain forward {
        # Zero Trust
        type filter hook forward priority 0; policy drop;
        ct state established,related accept

        #bLAN a accès à la DMZ et à Internet
        iifname "enp0s9" accept

        # La DMZ a accès uniquement à Internet
        iifname "enp0s8" oifname "enp0s3" accept

        log prefix "[NFT-BLOCKED]"
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # Masquage de l'IP (NAT) pour l'accès Internet // reddit
        oifname "enp0s3" masquerade
    }
}
```

**2. Application de la politique de sécurité :**

```bash
sudo nft flush ruleset
sudo nft -f /etc/nftables.conf
```
*Validation : Le `ping` depuis le LAN vers la DMZ est fonctionnel, mais le `ping` depuis la DMZ vers le LAN est désormais bloqué par le pare-feu. si vous voulez tester si ça marche c'est le test à faire*

## Étape 5 : DNAT (portforwading)

**Firewall**
```yaml
#!/usr/sbin/nft -f
flush ruleset

# --- TABLE DE FILTRAGE (Sécurité) ---
table inet filter {
    chain forward {
        # 1. Politique Zero Trust : tout est bloqué par défaut
        type filter hook forward priority 0; policy drop;

        ct state established,related accept

        iifname "enp0s9" accept

        iifname "enp0s8" oifname "enp0s3" accept
        #Nouvelle ligne pour le portforwading du port 80 (http)
        iifname "enp0s3" oifname "enp0s8" ip daddr 10.10.10.10 tcp dport 80 accept
        
        log prefix "[NFT-BLOCKED] "
    }
}

# --- TABLE NAT (Routage et Redirection) ---
table ip nat {
    # [NOUVEAU] Redirection du trafic entrant (DNAT)
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        
        # Tout ce qui arrive sur le port 80 WAN est redirigé vers l'IP de la DMZ
        iifname "enp0s3" tcp dport 80 dnat to 10.10.10.10
    }

    # Masquage de l'IP (SNAT) pour sortir sur Internet
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp0s3" masquerade
    }
}
```

**commandes pour remettre le Firewall en état**
```bash
sudo nft flush ruleset
sudo nft -f /etc/nftables.conf
```
*Memo : Le Firewall à du mal à s'activer refait cette expérience sur machine wiped pour voir comment fix.
