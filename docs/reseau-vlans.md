# 🌐 Architecture Réseau et Plan d'Adressage (VLANs)

> **Objectif :** segmenter l'infrastructure par niveau de criticité (Management, Serveurs, Postes, Attaquant) avec filtrage inter-VLAN centralisé sur **OPNsense**.

## 📑 Sommaire

1. [Dimensionnement des VLANs](#1-dimensionnement-des-vlans)
2. [Plan d'adressage IP des machines](#2-plan-dadressage-ip-des-machines)
3. [Notes d'architecture et points de contrôle](#3-notes-darchitecture-et-points-de-contrôle)

---

## 1. Dimensionnement des VLANs

| ID VLAN | Nom du VLAN | Interface OPNsense | Sous-réseau | Passerelle (OPNsense) | Serveur DHCP | Plage DHCP | Rôle et périmètre |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 10 | Management | `vtnet1.10` | `10.10.10.0/24` | `10.10.10.1` | Désactivé (IP fixes) | — | Administration Proxmox, OPNsense, Wazuh, Zabbix/Grafana |
| 20 | Serveurs | `vtnet1.20` | `10.10.20.0/24` | `10.10.20.1` | Désactivé (IP fixes) | — | Contrôleur de domaine (AD/DNS/DHCP), Services Web (GLPI) |
| 30 | Postes | `vtnet1.30` | `10.10.30.0/24` | `10.10.30.1` | Windows Server (AD) | `10.10.30.100 – .200` | Postes clients utilisateurs joints au domaine (Windows 10/11) |
| 40 | Attaquant | `vtnet1.40` | `10.10.40.0/24` | `10.10.40.1` | OPNsense | `10.10.40.100 – .150` | Machine d'attaque (Kali Linux) et cible web isolée (DVWA) |
| — | VPN WireGuard | `wg0` | `10.10.99.0/24` | `10.10.99.1` | — | Attribution statique | Accès distant administrateur vers le VLAN Management |

---

## 2. Plan d'adressage IP des machines

| Machine | Type | VLAN | Adresse IP | Masque | Passerelle | DNS primaire |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- |
| **OPNsense (WAN)** | `VM` | WAN (`vmbr0`) | DHCP box/routeur | `/24` | Box/Routeur | Box / Quad9 |
| **Hôte Proxmox VE** | `Hôte` | VLAN 10 (`vmbr1.10`) | `10.10.10.2` | `255.255.255.0` | `10.10.10.1` | `10.10.10.1` |
| **Wazuh (SIEM)** | `VM` | VLAN 10 | `10.10.10.10` | `255.255.255.0` | `10.10.10.1` | `10.10.20.10` |
| **Zabbix + Grafana** | `LXC` | VLAN 10 | `10.10.10.11` | `255.255.255.0` | `10.10.10.1` | `10.10.20.10` |
| **Windows Server 2022 (AD)** | `VM` | VLAN 20 | `10.10.20.10` | `255.255.255.0` | `10.10.20.1` | `127.0.0.1` (puis `.1`) |
| **Services Web (GLPI)** | `LXC` | VLAN 20 | `10.10.20.20` | `255.255.255.0` | `10.10.20.1` | `10.10.20.10` |
| **Client Windows 10/11** | `VM` | VLAN 30 | DHCP AD (`.100+`) | `255.255.255.0` | `10.10.30.1` | `10.10.20.10` |
| **Kali Linux** | `VM` | VLAN 40 | `10.10.40.10` | `255.255.255.0` | `10.10.40.1` | `10.10.40.1` |
| **Cible Web (DVWA)** | `LXC` | VLAN 40 | `10.10.40.20` | `255.255.255.0` | `10.10.40.1` | `10.10.40.1` |
| **Client Admin (WireGuard)** | `Tunnel` | VPN (`wg0`) | `10.10.99.2` | `255.255.255.255` | `10.10.99.1` | `10.10.20.10` |

---

## 3. Notes d'architecture et points de contrôle

> [!IMPORTANT]
> **Relais DHCP (VLAN 30 → VLAN 20) :** Le serveur DHCP étant hébergé sur Windows Server (`10.10.20.10`), le service **DHCP Relay** doit être activé sur l'interface `vtnet1.30` d'OPNsense pour relayer les requêtes des postes clients.

> [!CAUTION]
> **Isolation du VLAN 40 :** L'interface `vtnet1.40` doit comporter une règle de blocage explicite vers les réseaux privés (`RFC1918`) afin d'empêcher tout accès depuis la machine d'attaque (Kali) ou la cible vulnérable (DVWA) vers les VLANs 10, 20, 30 et le réseau WAN local.
