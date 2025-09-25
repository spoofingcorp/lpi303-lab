# Installation de Suricata sur Ubuntu Server 24.04

Ce document décrit comment :

1. Installer Suricata sur Ubuntu Server 24.04.
2. Télécharger et activer les règles communautaires (Emerging Threats / ET Open).
3. Simuler depuis une VM attaquante des scans (Nmap) afin de déclencher des alertes Suricata.

---

## 1) Pré‑requis et architecture de test

1. **Machine IDS** : Ubuntu Server 24.04, interface réseau connectée au segment à surveiller (ex. `eth0/ensXX`).
2. **VM attaquante** : VM Ubuntu ou Kali placée sur le même réseau ou routée pour que le trafic traverse l'interface surveillée.
3. S'assurer que la machine IDS voit le trafic (mode pont / mirror selon l'hyperviseur). Sur Proxmox/VirtualBox/ESXi : configurer l'interface en **bridged** ou activer le **promiscuous mode** sur le bridge/switch virtuel.

> Remarque : Suricata fournit `suricata-update` pour gérer les rules (notamment ET Open). 
---

## 2) Installation de Suricata

```bash
# Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# Installer Suricata et suricata-update
sudo apt install -y suricata suricata-update

# Vérifier la version
suricata -V
```

*Si vous souhaitez la dernière version upstream, envisagez l'installation depuis les paquets officiels de l'OISF ou la compilation depuis les sources (procédure non couverte ici).*

---

## 3) Configuration de l'interface d'écoute

1. Identifier l'interface à surveiller :

```bash
ip -br link
```

2. Démarrage manuel (test) sur `eth0` :

```bash
sudo suricata -c /etc/suricata/suricata.yaml -i eth0
```

3. Utilisation systemd :

```bash
# (optionnel) indiquer l'interface dans /etc/default/suricata :
# INTERFACES="eth0"
sudo systemctl enable --now suricata.service
sudo systemctl status suricata
```

> Selon votre besoin, vous pouvez configurer Suricata pour utiliser `af-packet`, `nfqueue`, `pfring`, etc. Ces modes se règlent dans `suricata.yaml` (section capture).

---

## 4) Télécharger et activer les règles communautaires (ET Open)

```bash
# Mettre à jour la base de rules via suricata-update
sudo suricata-update

# Vérifier le dossier des règles
ls -lah /var/lib/suricata/rules

# Redémarrer Suricata pour appliquer les règles
sudo systemctl restart suricata
```

`suricata-update` télécharge et installe les bundles de rules (par défaut ET Open quand disponible) et génère le fichier `suricata.rules`.

---

## 5) Vérifier les logs et alerts (format JSON)

1. Vérifier la configuration d'`eve.json` dans `/etc/suricata/suricata.yaml` (généralement activé par défaut).
2. Sur l'IDS, visualiser les alertes en temps réel :

```bash
sudo apt install -y jq
sudo tail -F /var/log/suricata/eve.json | jq -C 'select(.event_type=="alert")'
```

Vous pourrez filtrer ensuite par `src_ip`, `dest_ip`, `signature_id`, etc.

---

## 6) Simulation d'un attaquant — commandes Nmap depuis la VM attaquante

Remplacez `192.168.1.10` par l'IP de la cible surveillée.

```bash
# 1) SYN scan (rapide)
sudo nmap -sS -T4 -p 1-1024 192.168.1.10

# 2) Scan de service + version
sudo nmap -sV -p 22,80,443 192.168.1.10

# 3) Scan complet de ports TCP (lent)
sudo nmap -p- -T3 192.168.1.10

# 4) Scan agressif (OS detect, scripts)
sudo nmap -A -T4 192.168.1.10

# 5) Scan furtif (low rate)
sudo nmap -sS -T1 192.168.1.10

# 6) Scan UDP (bruyant / moins fiable)
sudo nmap -sU -p 53,123 192.168.1.10
```

Après chaque exécution, observez `/var/log/suricata/eve.json` sur l'IDS pour vérifier les alertes générées.

---

## 7) Exemple de règle locale pour détecter des SYN scans

Créez un fichier `/etc/suricata/rules/local-nmap.rules` :

```suricata
alert tcp any any -> any any (msg:"LOCAL Possible SYN scan"; flags:S; threshold: type both, track by_src, count 20, seconds 60; sid:1000001; rev:1;)
```

Cette règle génère une alerte si une source envoie plus de 20 SYN en 60 secondes (ajustez selon votre réseau).

Pour activer la règle locale :
- soit inclure le fichier dans `suricata.yaml` sous `rule-files:` ;
- soit gérer via votre flux `suricata-update` (prendre garde à l'écrasement des fichiers locaux si vous n'avez pas correctement configuré la section `manage-configs`).

---

## 8) Workflow de tests et validation

1. Lancer sur l'IDS :

```bash
sudo tail -F /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'
```

2. Depuis la VM attaquante, lancer un ou plusieurs scans Nmap ci‑dessus.
3. Sur l'IDS, identifier l'alerte (sid, catégorie, src_ip, dest_ip).
4. Ajuster les règles et seuils pour réduire les faux positifs.

---

## 9) Dépannage fréquent

- **Aucune alerte lors d’un scan** : vérifier que l'IDS voit le trafic (interface en mode promiscuous, moteur de capture configuré correctement) et que Suricata écoute la bonne interface.
- **Règles manquantes / erreurs de téléchargement** : exécuter `sudo suricata-update` en mode verbeux et vérifier la connectivité. En cas de problème avec le téléchargement automatique, récupérer manuellement le bundle ET Open.


---

## Annexes — commandes utiles

```bash
# Vérifier l'état du service
sudo systemctl status suricata

# Voir les règles chargées
suricata -T -c /etc/suricata/suricata.yaml

# Emplacement des règles
ls -lah /var/lib/suricata/rules

# Visualiser eve.json
sudo jq -C . /var/log/suricata/eve.json | less -R
```

---


