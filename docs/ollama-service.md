# Configuration Ollama — Service Systemd

## Service principal

Fichier : `/etc/systemd/system/ollama.service`

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

[Install]
WantedBy=default.target
```

## Override — Exposition réseau

Fichier : `/etc/systemd/system/ollama.service.d/override.conf`

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

> Sans cette variable, Ollama écoute uniquement sur 127.0.0.1 (localhost)
> et n'est pas accessible depuis le réseau local ni depuis Windows.

## Commandes de gestion

```bash
# Créer/éditer le fichier override
sudo systemctl edit ollama.service

# Recharger la configuration systemd
sudo systemctl daemon-reload

# Redémarrer le service
sudo systemctl restart ollama

# Vérifier le statut
sudo systemctl status ollama

# Voir les logs en temps réel
sudo journalctl -u ollama -f

# Vérifier le port d'écoute
ss -tlnp | grep 11434
```

## Résultat attendu après configuration

```bash
ss -tlnp | grep 11434
LISTEN 0  4096  *:11434  *:*
#              ↑ écoute sur toutes les interfaces
```

## Test depuis Windows PowerShell

```powershell
Invoke-WebRequest -Uri "http://192.168.1.100:11434/api/tags" -Method GET
# StatusCode : 200 → API accessible ✅
```