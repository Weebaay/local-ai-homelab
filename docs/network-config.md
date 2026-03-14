# Configuration réseau — IP fixe via Netplan

## Fichier : /etc/netplan/50-cloud-init.yaml

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.254
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

## Commandes

```bash
# Appliquer la configuration
sudo netplan apply

# Vérifier l'IP
ip a

# Vérifier la route par défaut
ip route

# Tester la connectivité
ping -c 4 8.8.8.8
ping -c 4 google.com
```

## Explication des paramètres

| Paramètre | Valeur | Description |
|---|---|---|
| `dhcp4: no` | false | Désactive l'attribution automatique d'IP |
| `addresses` | 192.168.1.100/24 | IP fixe du serveur + masque /24 |
| `via` | 192.168.1.254 | Gateway (routeur) du réseau local |
| `nameservers` | 8.8.8.8 / 1.1.1.1 | DNS Google et Cloudflare |

## Trouver sa gateway (Windows hôte)

```cmd
ipconfig
# Chercher : "Passerelle par défaut" sur la carte réseau active
```