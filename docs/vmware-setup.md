# Configuration VM VMware Workstation

## Paramètres de la VM

| Paramètre | Valeur | Justification |
|---|---|---|
| Nom | `srv-ia` | Nom explicite de type serveur |
| OS | Ubuntu 64-bit | Compatible Ubuntu Server 24.04 |
| RAM | 6144 Mo (6 Go) | Suffisant pour Ollama + Qwen2.5 7B |
| CPU | 2 cœurs | Minimum pour l'inférence CPU |
| Disque | 30 Go | OS (5 Go) + Modèle IA (~5 Go) + marge |
| Allocation disque | Dynamique | Grandit selon l'usage réel |
| Réseau | **Bridged** | IP propre sur le réseau local |

## Mode réseau Bridged vs NAT

| Mode | Description | Usage |
|---|---|---|
| **Bridged** ✅ | La VM a sa propre IP sur le réseau local | Administration SSH, accès API |
| NAT | La VM partage l'IP de l'hôte | Accès internet simple, SSH compliqué |
| Host-only | La VM ne voit que l'hôte | Tests isolés, pas d'internet |

> Le mode Bridged est indispensable pour accéder à l'API Ollama (port 11434)
> depuis Windows sans configuration NAT complexe.

## Consommation RAM estimée

| Composant | RAM utilisée |
|---|---|
| Windows hôte | ~4 Go |
| VM srv-ia (idle) | ~1 Go |
| Ollama + Qwen2.5 7B (actif) | ~5-6 Go |
| Open WebUI (Windows) | ~500 Mo |
| **Total en utilisation** | **~11-12 Go** |

> Avec 16 Go de RAM, l'IA et les VMs de lab ne peuvent pas tourner simultanément.
> Éteindre srv-ia pendant les sessions lab VMware libère ~7 Go.

## Modèles selon la RAM disponible pour la VM

| Modèle | RAM VM recommandée | Qualité |
|---|---|---|
| phi3:mini | 4 Go | Correct, très rapide |
| qwen2.5:7b | 6 Go | Très bon ✅ |
| llama3.1:8b | 7 Go | Excellent |
| llama3:70b | 40+ Go | Hors portée sans GPU |