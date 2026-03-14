# 🤖 Local AI Homelab — Serveur IA sur VM Ubuntu Server

> Déploiement d'un serveur IA local et privé sous **Ubuntu Server 24.04 LTS** dans une VM **VMware Workstation**, accessible via **SSH** depuis Windows et exposant une API REST pour des interfaces comme **Open WebUI**.

---

## 📋 Table des matières

- [Aperçu du projet](#-aperçu-du-projet)
- [Architecture](#-architecture)
- [Prérequis](#-prérequis)
- [Installation étape par étape](#-installation-étape-par-étape)
  - [1. Création de la VM VMware](#1-création-de-la-vm-vmware)
  - [2. Installation Ubuntu Server 24.04](#2-installation-ubuntu-server-2404)
  - [3. Configuration réseau (IP fixe)](#3-configuration-réseau-ip-fixe)
  - [4. Accès SSH depuis Windows](#4-accès-ssh-depuis-windows)
  - [5. Installation d'Ollama](#5-installation-dollama)
  - [6. Téléchargement du modèle IA](#6-téléchargement-du-modèle-ia)
  - [7. Exposition de l'API sur le réseau](#7-exposition-de-lapi-sur-le-réseau)
  - [8. Interface Open WebUI sur Windows](#8-interface-open-webui-sur-windows)
- [Compétences démontrées](#-compétences-démontrées)
- [Résolution de problèmes](#-résolution-de-problèmes)
- [Références](#-références)

---

## 🎯 Aperçu du projet

Ce projet consiste à déployer un **assistant IA local et privé** sur un serveur Ubuntu Server virtualisé, sans dépendance au cloud. L'objectif est de disposer d'un agent IA personnel :

- 🔒 **100% privé** — aucune donnée envoyée à des services tiers
- 💻 **Accessible depuis Windows** via SSH et navigateur web
- ⚙️ **Administré en ligne de commande** comme un vrai serveur
- 🔄 **Service persistant** grâce à systemd (redémarrage automatique)

**Stack technique :**

| Composant | Technologie | Rôle |
|---|---|---|
| Hyperviseur | VMware Workstation | Virtualisation |
| OS serveur | Ubuntu Server 24.04 LTS | Système sans GUI |
| Moteur IA | Ollama 0.17.7 | Exécution des modèles LLM |
| Modèle IA | Qwen2.5 7B | Inférence locale |
| Interface | Open WebUI 0.8.8 | UI type ChatGPT |
| Accès distant | SSH + OpenSSH | Administration distante |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│              PC Windows (Hôte)                  │
│                                                 │
│  ┌─────────────────┐    ┌─────────────────────┐ │
│  │  Windows Terminal│    │     Navigateur      │ │
│  │  SSH Client      │    │  http://localhost   │ │
│  └────────┬─────────┘    │       :8080         │ │
│           │ SSH           └──────────┬──────────┘ │
│           │ :22                      │ HTTP       │
│  ┌────────▼─────────────────────────▼──────────┐ │
│  │         VMware Workstation                   │ │
│  │  ┌───────────────────────────────────────┐  │ │
│  │  │     VM : srv-ia (Ubuntu Server)        │  │ │
│  │  │     IP fixe : 192.168.1.100            │  │ │
│  │  │                                        │  │ │
│  │  │  ┌─────────────────────────────────┐  │  │ │
│  │  │  │   Ollama (service systemd)       │  │  │ │
│  │  │  │   Port 11434 — 0.0.0.0          │  │  │ │
│  │  │  │   Modèle : Qwen2.5 7B (~5 Go)   │  │  │ │
│  │  │  └─────────────────────────────────┘  │  │ │
│  │  └───────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

---

## ⚙️ Prérequis

### Matériel recommandé
| Ressource | Minimum | Recommandé |
|---|---|---|
| RAM hôte | 16 Go | 32 Go |
| Stockage | 50 Go libres | 100 Go |
| CPU | 4 cœurs | 8 cœurs |

### Logiciels requis
- [VMware Workstation Pro/Player](https://www.vmware.com/products/workstation-pro.html)
- [Ubuntu Server 24.04 LTS ISO](https://ubuntu.com/download/server)
- Windows Terminal (recommandé pour SSH)
- Python 3.11+ (pour Open WebUI côté Windows)

---

## 📦 Installation étape par étape

### 1. Création de la VM VMware

Créer une nouvelle VM avec la configuration suivante :

| Paramètre | Valeur |
|---|---|
| Nom | `srv-ia` |
| OS | Ubuntu 64-bit |
| RAM | 6144 Mo (6 Go) |
| CPU | 2 cœurs |
| Disque | 30 Go (allocation dynamique) |
| Réseau | **Bridged** (mise en réseau reliée par un pont) |

> ⚠️ Le mode **Bridged** est essentiel : il attribue à la VM une IP sur le réseau local, permettant un accès SSH direct depuis l'hôte Windows.

---

### 2. Installation Ubuntu Server 24.04

Démarrer la VM avec l'ISO Ubuntu Server. Points importants lors de l'installation :

```
Type d'installation  → Ubuntu Server (pas Minimized)
Réseau               → Laisser DHCP par défaut (on fixera l'IP après)
Stockage             → Use entire disk
SSH                  → ✅ Cocher "Install OpenSSH server"
Snaps                → Ne rien sélectionner
```

> ✅ L'activation d'OpenSSH lors de l'installation est critique pour l'accès distant.

---

### 3. Configuration réseau (IP fixe)

Après le premier démarrage, identifier le nom de l'interface réseau :

```bash
ip a
# Repérer l'interface, généralement ens33
```

Identifier la gateway de ton réseau (sur Windows hôte) :
```cmd
ipconfig
# Chercher "Passerelle par défaut" sur la carte Wi-Fi/Ethernet active
```

Modifier la configuration netplan :

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.254   # ← Adapter selon ta gateway
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Appliquer la configuration :

```bash
sudo netplan apply
# La session SSH se coupe — normal, l'IP a changé
```

Vérifier la connectivité :

```bash
ping -c 4 8.8.8.8
```

---

### 4. Accès SSH depuis Windows

Depuis **Windows Terminal** sur l'hôte :

```powershell
ssh jeanpaul@192.168.1.100
```

En cas d'erreur "REMOTE HOST IDENTIFICATION HAS CHANGED" :

```powershell
ssh-keygen -R 192.168.1.100
```

> 💡 À partir de cette étape, **toute l'administration se fait en SSH**. La console VMware n'est plus nécessaire.

---

### 5. Installation d'Ollama

Mettre à jour le système puis installer Ollama :

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://ollama.com/install.sh | sh
```

Vérifier l'installation :

```bash
ollama --version
# ollama version is 0.17.7
```

Vérifier que le service systemd est actif :

```bash
sudo systemctl status ollama
# Active: active (running)
```

> Ollama est automatiquement configuré comme **service systemd** et démarre au boot de la VM.

---

### 6. Téléchargement du modèle IA

```bash
ollama pull qwen2.5:7b
# ~4.5 Go — patientez pendant le téléchargement
```

Lister les modèles disponibles :

```bash
ollama list
```

Tester le modèle en ligne de commande :

```bash
ollama run qwen2.5:7b
>>> Explique ce qu'est le protocole STP en réseau
# Ctrl+D pour quitter
```

**Modèles compatibles selon la RAM disponible :**

| Modèle | RAM requise | Usage recommandé |
|---|---|---|
| `phi3:mini` | ~3 Go | Léger, réponses rapides |
| `qwen2.5:7b` | ~5 Go | Bon équilibre qualité/perf ✅ |
| `llama3.1:8b` | ~6 Go | Excellent en raisonnement |
| `deepseek-r1:7b` | ~6 Go | Excellent pour le code |

---

### 7. Exposition de l'API sur le réseau

Par défaut, Ollama écoute uniquement sur `127.0.0.1`. Pour le rendre accessible depuis le réseau local :

```bash
sudo systemctl edit ollama.service
```

Dans l'éditeur, saisir **avant** les commentaires existants :

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

Sauvegarder (`Ctrl+X` → `Y` → `Entrée`) puis redémarrer :

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Vérifier que l'API écoute sur toutes les interfaces :

```bash
ss -tlnp | grep 11434
# LISTEN 0  4096  *:11434  *:*   ← Correct
```

Tester depuis Windows PowerShell :

```powershell
Invoke-WebRequest -Uri "http://192.168.1.100:11434/api/tags" -Method GET
# StatusCode : 200 → Succès ✅
```

---

### 8. Interface Open WebUI sur Windows

Installer Open WebUI sur Windows (se connecte à Ollama sur la VM) :

```powershell
# Installer Python 3.12 si nécessaire
winget install Python.Python.3.12

# Installer Open WebUI
pip install open-webui
```

Lancer Open WebUI en pointant vers la VM :

```powershell
$env:OLLAMA_BASE_URL="http://192.168.1.100:11434"; open-webui serve
```

Accéder à l'interface :

```
http://localhost:8080
```

> 🎉 L'interface ChatGPT est maintenant connectée au modèle Qwen2.5 qui tourne sur la VM Ubuntu Server.

---

## 🎓 Compétences démontrées

| Domaine | Compétences |
|---|---|
| **VMware / Virtualisation** | Création et configuration de VM, modes réseau (Bridged/NAT/Host-only), allocation dynamique de disque |
| **Linux / Ubuntu Server** | Installation sans GUI, gestion des paquets apt, arborescence Linux, permissions sudo |
| **Réseau** | Configuration IP statique via netplan, compréhension gateway/DNS, diagnostic ping/ss |
| **SSH** | Connexion distante depuis Windows, gestion des clés d'hôte, administration 100% CLI |
| **Systemd** | Gestion des services (status/restart/enable), fichiers unit, drop-in override |
| **API REST** | Test d'API avec PowerShell (Invoke-WebRequest), compréhension JSON, ports et protocoles HTTP |
| **Ollama / IA locale** | Déploiement de LLM en local, gestion des modèles, configuration réseau du service |

---

## 🔧 Résolution de problèmes

### La VM n'a pas accès à Internet
```bash
# Vérifier la gateway dans netplan
cat /etc/netplan/50-cloud-init.yaml

# Vérifier la route par défaut
ip route

# Tester la connectivité
ping 8.8.8.8
```
→ Vérifier que la `via` dans netplan correspond à la gateway de ton réseau (`ipconfig` sur Windows).

### Ollama n'est pas accessible depuis Windows
```bash
# Vérifier sur quel port Ollama écoute
ss -tlnp | grep 11434

# Si 127.0.0.1:11434 → la variable OLLAMA_HOST n'est pas appliquée
sudo systemctl cat ollama.service | grep OLLAMA_HOST
```

### La session SSH se coupe après netplan apply
C'est normal — l'IP a changé. Se reconnecter avec la nouvelle IP fixe :
```powershell
ssh jeanpaul@192.168.1.100
```

---

## 📚 Références

- [Documentation Ollama](https://ollama.com/docs)
- [Open WebUI GitHub](https://github.com/open-webui/open-webui)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [Netplan Documentation](https://netplan.io/reference)
- [Systemd Unit Files](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

---

## 👤 Auteur

**Jean-Paul** — Technicien Support IT N2 | En formation TSSR  
Passionné par l'administration systèmes, les réseaux et l'automatisation.

[![GitHub](https://img.shields.io/badge/GitHub-Profile-black?logo=github)](https://github.com/Weebaay)