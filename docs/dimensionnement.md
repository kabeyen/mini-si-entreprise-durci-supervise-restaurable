# 🖥️ Dimensionnement de l'infrastructure (Mini-SI)

> **Objectif :** faire tourner le cœur de l'infrastructure en continu avec une marge de sécurité d'environ **1 Go** sur l'hôte, et n'allumer les machines de test **qu'à la demande**.

## 📑 Sommaire

1. [Allocation des ressources](#1-allocation-des-ressources-prévu-vs-réel)
2. [Justification des choix techniques](#2-justification-des-choix-techniques)
3. [Scénarios d'exécution simultanée](#3-scénarios-dexécution-simultanée)

---

## 1. Allocation des ressources (Prévu vs Réel)

**Légende :** 🟢 Permanent (24/7) · 🟡 À la demande

| Machine | Type | vCPU | RAM prévue | RAM réelle | Disque prévu | Disque réel | Mode | Rôle |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **OPNsense** | `VM` | 2 | 1,5 Go | _À mesurer_ | 12 Go | _À mesurer_ | 🟢 24/7 | Pare-feu, routage inter-VLAN, IDS Suricata, VPN WireGuard |
| **Windows Server 2022** | `VM` | 2 | 3,0 Go | _À mesurer_ | 40 Go | _À mesurer_ | 🟢 24/7 | AD DS, DNS, DHCP, GPO, PKI (AD CS) |
| **Wazuh (All-in-One)** | `VM` | 2 | 4,0 Go | _À mesurer_ | 30 Go | _À mesurer_ | 🟢 24/7 | SIEM / XDR (Indexeur + Serveur + Dashboard) |
| **Services Web (GLPI)** | `LXC` | 1 | 1,5 Go | _À mesurer_ | 8 Go | _À mesurer_ | 🟢 24/7 | Debian 12, Nginx/Apache, MariaDB, GLPI |
| **Supervision** | `LXC` | 1 | 1,0 Go | _À mesurer_ | 6 Go | _À mesurer_ | 🟢 24/7 | Debian 12, Zabbix Server + Grafana |
| **Cible Web (DVWA)** | `LXC` | 1 | 0,5 Go | _À mesurer_ | 4 Go | _À mesurer_ | 🟡 À la demande | Debian 12, cible vulnérable (isolée) pour scénarios ATT&CK |
| **Client Windows 10/11** | `VM` | 2 | 3,0 Go | _À mesurer_ | 30 Go | _À mesurer_ | 🟡 À la demande | Poste utilisateur joint au domaine (tests GPO, Sysmon) |
| **Kali Linux** | `VM` | 2 | 2,0 Go | _À mesurer_ | 20 Go | _À mesurer_ | 🟡 À la demande | Machine d'attaque (scénarios Purple Team) |
| **Total infrastructure** | — | — | **16,5 Go** | — | **150 Go** | — | — | — |
| **Total permanent (H24)** | — | — | **11,0 Go** | — | **96 Go** | — | — | Cœur du SI actif en continu |

---

## 2. Justification des choix techniques

### 📦 Privilégier les conteneurs LXC (3 conteneurs)

Évite la surcharge d'un noyau complet par service Linux : économie d'environ **1,5 à 2 Go de RAM** par rapport à des VM classiques.

### 🔍 Wazuh à 4 Go

C'est le minimum recommandé pour éviter les crashs Java (`OutOfMemory`) de l'indexeur OpenSearch.

> [!TIP]
> **Plan B** si l'hôte sature : brider la JVM (`jvm.options`) à **1,5 Go**, passer la VM à **3 Go** et limiter la rétention des logs à **7 jours**.

### 🪟 Windows Server sans interface graphique (ou minimale)

3 Go suffisent pour AD DS, DNS et AD CS, tant que le serveur n'héberge pas d'applications lourdes.

---

## 3. Scénarios d'exécution simultanée

Hôte de **16 Go**, marge de sécurité de **~1 Go**, cœur permanent de **11 Go** : il reste donc **~4 Go** pour les machines à la demande.

| Scénario | Machines à la demande | RAM à la demande | Total avec le cœur | Faisable ? |
| :--- | :--- | :---: | :---: | :---: |
| Attaque simple | Kali + DVWA | 2,5 Go | 13,5 Go | ✅ |
| Test GPO / Sysmon | Client Windows | 3,0 Go | 14,0 Go | ✅ |
| Purple Team complet | Kali + DVWA + Client Windows | 5,5 Go | 16,5 Go | ❌ dépasse l'hôte |

> [!NOTE]
> Les valeurs « RAM réelle » et « Disque réel » seront complétées après mesure sur le labo.
