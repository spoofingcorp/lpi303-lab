# **LAB \- LPI 303 Sécurité** 

Ce guide fournit des exercices approfondis pour chaque module de la formation, avec des contextes d'entreprise, des commandes détaillées, des procédures de débogage et des bonnes pratiques. Toutes les procédures ont été vérifiées sur **Ubuntu Server 24.04 LTS**.

### **Environnement de base :**

* **Deux Machines Virtuelles (VM)** sous Ubuntu Server 24.04 :  
  * srv-main : Le serveur que nous allons sécuriser.  
  * srv-client : La machine "attaquante" pour les tests.

* (Un client Windows 11 est optionnel mais utile pour simuler un utilisateur final).  
* **Réseau :** Assurez-vous que vos VM peuvent communiquer entre elles (réseau "interne" ou "hôte privé" dans votre hyperviseur). Notez leurs adresses IP.

## **Module 1 : Cryptographie et Systèmes de Fichiers Chiffrés**

### **Atelier 1.1 : Gestion Avancée de GPG**

**Contexte d'Entreprise :** Une équipe de développeurs doit échanger des informations sensibles. Pour se conformer au RGPD, l'utilisation de services tiers est interdite. Vous mettez en place GPG pour chiffrer les fichiers et signer les commits Git, garantissant confidentialité et authenticité. \[Image d'une clé et d'un cadenas numérique\]

#### **Étapes Détaillées :**

1. **Génération d'une paire de clés (sur srv-main pour alice) :**  
   \# Créez l'utilisateur pour le test  
   sudo useradd \-m \-s /bin/bash alice  
   sudo passwd alice  
   su \- alice

   \# Lancez l'assistant interactif de création de clé  
   gpg \--full-generate-key

   * **Type de clé : (1) RSA and RSA**. Crée une clé principale pour la certification et une sous-clé pour le chiffrement.  
   * **Taille : 4096** bits. Un standard robuste.  
   * **Validité : 1y** (1 an). Forcer le renouvellement est une bonne pratique.  
   * **Identifiants :** Renseignez un nom (Alice Dev), une e-mail (alice@entreprise.com).  
   * **Phrase de passe :** Choisissez-en une très robuste. Elle est le dernier rempart protégeant votre clé privée.

2. **Explorer le trousseau de clés :**  
   \# Lister les clés publiques. Notez l'identifiant long (empreinte).  
   gpg \--list-keys \--keyid-format=long

   \# Lister les clés privées (secretes).  
   gpg \--list-secret-keys \--keyid-format=long

3. Exporter/Importer une clé publique :  
   Pour que Bob puisse écrire à Alice, il a besoin de sa clé publique.  
   \# Exporter la clé publique d'Alice dans un fichier texte (armor)  
   gpg \--armor \--export alice@entreprise.com \> alice\_pubkey.asc

   \# Simulez l'importation de cette clé par un autre utilisateur (ex: bob)  
   scp alice\_pubkey.asc bob@srv-client:\~/

\# Sur srv-client, en tant que bob:  
gpg \--import alice\_pubkey.asc

4. Vérifier et faire confiance à une clé (Web of Trust) :  
   Importer une clé ne suffit pas. Il faut s'assurer qu'elle appartient bien à la bonne personne.  
   \# Affichez l'empreinte de la clé importée  
   gpg \--fingerprint alice@entreprise.com

   \# Alice doit communiquer cette empreinte à Bob via un canal sécurisé (téléphone, face à face).  
   \# Si les empreintes correspondent, Bob peut signer la clé pour lui marquer sa confiance.

gpg \--edit-key [alice@entreprise.com](mailto:alice@entreprise.com)

\# Dans l'invite gpg\>, tapez 'sign', confirmez, puis 'save'.

5. **Chiffrement et Déchiffrement :**  
   \# (alice) Créez un fichier sensible  
   echo "API\_KEY=1234-ABCD-5678-EFGH" \> secrets.txt

   \# (alice) Chiffrez-le pour elle-même (ou pour un destinataire dont on a la clé publique)

gpg \--encrypt \--recipient alice@entreprise.com \--armor \-o secrets.asc secrets.txt

\# (alice) Déchiffrez le fichier. La phrase de passe de la clé privée sera demandée.

gpg \--decrypt secrets.asc \> secrets\_dechiffres.txt  
cat secrets\_dechiffres.txt

6. Signature et Vérification :  
   La signature prouve l'authenticité (l'auteur) et l'intégrité (pas de modification).

\# (alice) Crée une signature "détachée". Le fichier original n'est pas modifié.  
gpg \--detach-sign \--armor secrets.txt

\# (bob, sur srv-client) Pour vérifier, il a besoin du fichier original et de la signature  
scp secrets.txt secrets.txt.asc bob@srv-client:\~/

\# Sur srv-client, en tant que bob:  
gpg \--verify secrets.txt.asc secrets.txt 

\# La sortie "Good signature from 'Alice Dev'..." confirme l'authenticité.

### **Atelier 1.2 : Mise en place d'un Volume Chiffré avec LUKS**

**Contexte d'Entreprise :** Un serveur héberge des documents RH. Pour se prémunir contre le vol physique du serveur, la partition de données doit être chiffrée. Si un disque est volé, les données seront illisibles.

#### **Étapes Détaillées :**

**Prérequis :** Ajoutez un nouveau disque virtuel (ex: 8Go, /dev/sdb) à votre VM srv-main.

1. **Identifier et préparer le disque :**  
   \# Listez les périphériques bloc. Repérez votre nouveau disque (ex: sdb).  
   lsblk \-f

   \# ATTENTION : La commande suivante est DESTRUCTIVE.  
   \# Effacez toute signature de système de fichiers existante.  
   sudo wipefs \-a /dev/sdb

2. **Créer le conteneur LUKS :**  
   \# Formatez le disque avec LUKS2 (par défaut sur Ubuntu 24.04)  
   \# \--verbose pour plus d'infos, \--verify-passphrase pour éviter les typos.  
   sudo cryptsetup luksFormat \--type luks2 \--verbose \--verify-passphrase /dev/sdb  
   \# Répondez YES en majuscules, puis entrez une phrase de passe très robuste.

3. Vérifier et sauvegarder l'en-tête LUKS :  
   L'en-tête contient les informations vitales pour le déchiffrement. Sans lui, les données sont perdues.  
   \# Affichez les informations de l'en-tête (slots de clés, UUID, etc.)  
   sudo cryptsetup luksDump /dev/sdb

   \# SAUVEGARDEZ L'EN-TÊTE SUR UN SUPPORT EXTERNE SÉCURISÉ \!  
   sudo cryptsetup luksHeaderBackup /dev/sdb \--header-backup-file /root/sdb\_luks\_header.bkp

4. **Ouvrir, formater et monter le volume :**  
   \# Ouvrez le conteneur. Cela crée un périphérique mappé déchiffré.  
   sudo cryptsetup luksOpen /dev/sdb volume\_rh  
   \# Entrez votre phrase de passe.

   \# Créez un système de fichiers ext4 sur le volume déchiffré.  
   sudo mkfs.ext4 /dev/mapper/volume\_rh

   \# Montez-le  
   sudo mkdir \-p /srv/data\_rh  
   sudo mount /dev/mapper/volume\_rh /srv/data\_rh  
   sudo chown $USER:$USER /srv/data\_rh

5. **Utilisation et automatisation du montage :**  
   \# Vérifiez que le montage est présent et fonctionnel  
   df \-hT | grep volume\_rh  
   touch /srv/data\_rh/test\_confidentiel.txt

   Pour un montage automatique au démarrage (serveur non-interactif) :  
   1. **Créez un fichier clé :** sudo dd if=/dev/urandom of=/root/keyfile bs=1024 count=4  
   2. **Ajoutez la clé au conteneur LUKS :** sudo cryptsetup luksAddKey /dev/sdb /root/keyfile  
   3. **Configurez /etc/crypttab** pour ouvrir le volume au démarrage en utilisant la clé.  
   4. **Configurez /etc/fstab** pour monter le volume une fois ouvert.

   

   

   

6. **Sécuriser le volume (fermeture manuelle) :**

sudo umount /srv/data\_rh  
sudo cryptsetup luksClose volume\_rh

\# Le périphérique /dev/mapper/volume\_rh disparaît. Les données sont à nouveau inaccessibles.

lsblk \-f

## 

## **Module 2 : Contrôle d'Accès**

### **Atelier 2.1 : Manipulation des Permissions Avancées (SGID, Sticky Bit, ACLs)**

**Contexte d'Entreprise :** Vous administrez un serveur de développement avec un répertoire partagé /srv/projets/alpha. Les contraintes sont strictes : collaboration au sein du groupe devs, protection des fichiers de chaque utilisateur, et accès en lecture seule pour une stagiaire.

#### **Étapes Détaillées :**

1. **Mise en place initiale :**  
   \# Création du groupe, des utilisateurs et définition des mots de passe  
   sudo groupadd devs  
   sudo useradd \-m \-s /bin/bash \-G devs alice  
   sudo useradd \-m \-s /bin/bash \-G devs bob  
   sudo useradd \-m \-s /bin/bash emma  
   sudo passwd alice && sudo passwd bob && sudo passwd emma

   \# Création du répertoire de projet et attribution au groupe 'devs'  
   sudo mkdir \-p /srv/projets/alpha  
   sudo chown root:devs /srv/projets/alpha

2. **Application des bits spéciaux :**  
   \# 2 \= SGID (force l'héritage du groupe 'devs')  
   \# 1 \= Sticky Bit (seuls les propriétaires de fichiers peuvent les supprimer)  
   \# 770 \= rwx pour root, rwx pour le groupe devs, \--- pour les autres.  
   \# La commande 'chmod' combine les permissions : 2770 pour SGID, \+1000 pour Sticky Bit \-\> 3770  
   sudo chmod 3770 /srv/projets/alpha

   \# Vérification : la sortie doit être 'drwxrws--T'.  
   \# 's' dans les permissions du groupe indique le SGID.  
   \# 'T' (majuscule) indique le Sticky Bit sur un répertoire sans droit d'exécution pour 'autres'.  
   ls \-ld /srv/projets/alpha

3. **Test de comportement (très important) :**  
   \# En tant qu'admin, se connecter comme alice et créer un fichier  
   sudo su \- alice \-c "touch /srv/projets/alpha/rapport\_alice.txt"  
   \# Vérifier que le fichier appartient bien à 'alice:devs'  
   ls \-l /srv/projets/alpha/

   \# Se connecter comme bob et tenter de supprimer le fichier d'alice (doit échouer)  
   sudo su \- bob \-c "rm /srv/projets/alpha/rapport\_alice.txt"  
   \# La sortie doit être "rm: cannot remove '...': Operation not permitted"

   \# Bob peut créer son propre fichier  
   sudo su \- bob \-c "touch /srv/projets/alpha/notes\_bob.txt"

4. Mise en place de l'ACL pour un accès granulaire :  
   Les ACLs (Access Control Lists) outrepassent les permissions UNIX de base.  
   \# Installation de l'outil et vérification que le système de fichiers les supporte (ext4 le fait par défaut)  
   sudo apt update && sudo apt install acl \-y

   \# Création du fichier par l'admin  
   sudo touch /srv/projets/alpha/planning.md  
   sudo chown root:devs /srv/projets/alpha/planning.md  
   sudo chmod 640 /srv/projets/alpha/planning.md

   \# Application de l'ACL : modifier (-m) pour l'utilisateur (u) emma, donner la permission de lecture (r)  
   sudo setfacl \-m u:emma:r /srv/projets/alpha/planning.md

5. **Vérification et test de l'ACL :**  
   \# 'getfacl' montre les permissions de base ET les entrées ACL.  
   \# Vous verrez une nouvelle ligne "user:emma:r--"  
   getfacl /srv/projets/alpha/planning.md

   \# La commande 'ls \-l' montre un '+' à la fin des permissions pour indiquer qu'une ACL est active.  
   ls \-l /srv/projets/alpha/planning.md

   \# Test final en tant qu'emma  
   sudo su \- emma \-c "cat /srv/projets/alpha/planning.md" \# Doit réussir  
   sudo su \- emma \-c "echo 'modif' \>\> /srv/projets/alpha/planning.md" \# Doit échouer avec "Permission denied"

### 

### **Atelier 2.2 : Introduction à AppArmor**

**Contexte d'Entreprise :** Un serveur web Nginx est déployé. La politique "zero trust" impose le confinement strict des services. Le processus Nginx ne doit lire/écrire que dans son répertoire web (/var/www/html) et ses fichiers de logs/configuration. Toute autre action doit être bloquée et auditée.

#### **Étapes Détaillées :**

1. **Installation et vérification de l'état :**  
   sudo apt install nginx apparmor-utils \-y

   \# Vérifie que le service AppArmor est actif et chargé  
   sudo systemctl status apparmor

   \# Liste les profils AppArmor chargés et leur mode (enforce ou complain)  
   \# Cherchez la ligne contenant '/usr/sbin/nginx'. Elle doit être en mode 'enforce'.  
   sudo aa-status

2. Provoquer une violation de politique :  
   Nous allons configurer Nginx pour lire des fichiers depuis un emplacement non autorisé.  
   \# Création d'un site web de test dans un emplacement interdit par le profil par défaut  
   sudo mkdir /tmp/test-site  
   sudo sh \-c 'echo "Page web non autorisee" \> /tmp/test-site/index.html'

   \# Modification de la configuration Nginx  
   sudo nano /etc/nginx/sites-available/default  
   \# Changez la ligne 'root /var/www/html;' en 'root /tmp/test-site;'.

3. **Observer le blocage et analyser les logs :**  
   \# Testez la syntaxe Nginx et rechargez  
   sudo nginx \-t && sudo systemctl reload nginx

   \# Essayez d'accéder à la page. Vous obtiendrez une erreur 403 Forbidden.  
   curl localhost

   \# Analysez les logs d'audit du système pour voir la décision d'AppArmor  
   \# Cherchez les messages DENIED liés au profil nginx dans les logs récents  
   sudo journalctl \-k | grep "apparmor=\\"DENIED\\"" | grep "nginx"  
   \# La sortie montrera clairement l'opération (open), le profil (/usr/sbin/nginx), le nom du fichier (/tmp/test-site/index.html) et le PID du processus.

4. Utiliser le mode "Complain" pour le débogage de profil :  
   Ce mode est crucial pour développer ou ajuster un profil sans casser une application. Il ne bloque pas les actions, mais il continue de les logger.  
   \# Passer le profil nginx en mode complain  
   sudo aa-complain /etc/apparmor.d/usr.sbin.nginx  
   sudo aa-status | grep nginx 

\# Vérifiez que le mode a changé

\# Réessayez d'accéder à la page. Cette fois, ça devrait fonctionner \!  
curl localhost

\# Vérifiez à nouveau les logs. Vous verrez les mêmes messages d'audit, mais ils n'auront pas provoqué de blocage.

\# Outils pour faciliter la création de profils :  
sudo aa-genprof /usr/sbin/nginx  
\# Cet outil analyse les logs et vous propose interactivement de nouvelles règles.

5. **Restauration :**  
   \# Remettre le profil en mode blocage (enforce)  
   sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx

   \# N'oubliez pas de restaurer la configuration originale de Nginx \!

## 

## **Module 3 : Sécurité des Applications et Services**

### **Atelier 3.1 : Verrouillage d'OpenSSH et configuration de Fail2Ban**

**Contexte d'Entreprise :** Pour se conformer aux standards PCI-DSS et réduire la surface d'attaque, tous les serveurs doivent interdire la connexion root, forcer l'authentification par clé SSH et bannir automatiquement les IPs qui tentent des attaques par force brute. \[Image d'un bouclier protégeant un serveur\]

#### **Étapes Détaillées :**

1. **Prérequis CRUCIAL : Assurer son propre accès par clé \!**  
   Si vous désactivez l'authentification par mot de passe sans avoir de clé fonctionnelle, vous serez définitivement bloqué.  
   \# Sur votre machine LOCALE (pas la VM), générez une clé si vous n'en avez pas. ed25519 est moderne et performant.  
   ssh-keygen \-t ed25519 \-C "votre\_email@example.com"

   \# Copiez votre clé publique sur la VM srv-main (remplacez \<user\> et \<ip\_vm\>)  
   ssh-copy-id \<user\>@\<ip\_srv-main\>

   \# Testez la connexion. Vous devez pouvoir vous connecter SANS mot de passe.  
   ssh \<user\>@\<ip\_srv-main\>

2. Durcissement du fichier de configuration /etc/ssh/sshd\_config :  
   Sur srv-main, faites toujours une sauvegarde avant de modifier.  
   sudo cp /etc/ssh/sshd\_config /etc/ssh/sshd\_config.bak  
   sudo nano /etc/ssh/sshd\_config

   Trouvez et modifiez ces lignes pour qu'elles correspondent à :  
   PermitRootLogin no  
   PubkeyAuthentication yes  
   PasswordAuthentication no  
   ChallengeResponseAuthentication no  
   UsePAM no 

\# UsePAM no peut améliorer la sécurité en désactivant des modules d'authentification potentiellement faibles.

3. **Validation et application de la configuration :**  
   \# Vérifie la syntaxe du fichier AVANT d'appliquer. Si aucune sortie, c'est OK.  
   sudo sshd \-t

   \# Applique la nouvelle configuration  
   sudo systemctl restart sshd

   **Test :** OUVREZ UN NOUVEAU TERMINAL et essayez de vous connecter. Si ça fonctionne, vous pouvez fermer l'ancienne session.

4. Installation et configuration de Fail2Ban :

Fail2Ban analyse les logs et bannit les IPs malveillantes via des règles de pare-feu.  
sudo apt install fail2ban \-y

\# Ne modifiez JAMAIS jail.conf. Créez un fichier local pour vos surcharges.  
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local  
sudo nano /etc/fail2ban/jail.local

Dans ce fichier, cherchez la section \[sshd\] et assurez-vous que enabled est à true. Vous pouvez aussi personnaliser le temps de bannissement (bantime) et le nombre d'essais (maxretry).  
\[sshd\]  
enabled \= true  
port    \= ssh  
bantime \= 1h  
maxretry \= 3

5. **Vérification et test de Fail2Ban :**  
   \# Activez et démarrez le service  
   sudo systemctl enable \--now fail2ban

   \# Vérifie l'état de la "prison" sshd. Au début, 0 IP bannie.  
   sudo fail2ban-client status sshd

   \# Test : depuis srv-client, tentez de vous connecter 3-4 fois avec un mauvais utilisateur.  
   ssh nimportequoi@\<ip\_de\_srv-main\>  
   \# La connexion va soudainement refuser ("Connection refused" ou timeout).

   \# Retournez sur srv-main et revérifiez le statut  
   sudo fail2ban-client status sshd  
   \# Vous devriez voir l'IP de srv-client dans la liste des bannis.

   \# Pour débannir une IP manuellement :  
   sudo fail2ban-client set sshd unbanip \<ip\_a\_debannir\>

## **Module 4 : Sécurité Opérationnelle**

### **Atelier 4.1 : Détection d'Intrusion sur l'Hôte (HIDS) avec AIDE**

**Contexte d'Entreprise :** Un serveur web critique doit être surveillé pour toute modification de fichier non autorisée. AIDE (Advanced Intrusion Detection Environment) va créer une "photographie" de l'état des fichiers système importants pour détecter toute déviation.

#### **Étapes Détaillées :**

1. **Installation et configuration :**

sudo apt install aide \-y

\# Le fichier /etc/aide/aide.conf définit les règles. La configuration par défaut est un bon début.  
\# Vous pouvez l'adapter pour surveiller des répertoires spécifiques comme /var/www.

2. Initialisation de la base de données de référence :  
   Cette étape doit être réalisée sur un système fraîchement installé et considéré comme "propre".  
   \# Cette commande scanne le système et peut prendre plusieurs minutes.  
   sudo aideinit

   \# Elle crée /var/lib/aide/aide.db.new.gz. Pour l'activer, renommez-la.  
   sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

   **Bonne pratique :** Copiez aide.db.gz sur un support en lecture seule (CD-ROM, stockage immuable) pour qu'un attaquant ne puisse pas la modifier pour cacher ses traces.  
3. **Simulation d'une modification du système :**  
   \# Un attaquant pourrait ajouter un utilisateur ou modifier un binaire.

sudo useradd attaquant  
sudo chmod 777 /etc/hosts

4. **Exécution d'une vérification d'intégrité :**  
   \# Compare l'état actuel du système à la base de données de référence.

sudo aide \--check

Le rapport sera très détaillé et signalera :

* Fichiers ajoutés : /etc/passwd, /etc/shadow (lignes ajoutées), /home/attaquant.  
  * Fichiers modifiés : ATTRIBUTE CHANGES: /etc/hosts (Permissions modifiées).


5. **Automatisation et gestion des mises à jour légitimes :**  
   \# Pour automatiser la vérification (ex: tous les jours à 4h), éditez la crontab root  
   sudo crontab \-e

\# Ajoutez : 0 4 \* \* \* /usr/bin/aide \--check | mail \-s "Rapport AIDE pour $(hostname)" admin@example.com

\# Si une modification est légitime (ex: mise à jour de paquet), vous devez mettre à jour la base de référence.

sudo aide \--update

\# Cela crée un nouveau aide.db.new.gz. Examinez les changements puis remplacez l'ancienne base.

sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

## 

## **Module 5 : Sécurité Réseau et Évaluation des Vulnérabilités**

### **Atelier 5.1 : Audit de Vulnérabilités avec Nmap**

**Contexte d'Entreprise :** En tant qu'auditeur, vous devez scanner le serveur srv-main depuis srv-client pour découvrir sa surface d'attaque réseau, identifier les services exposés et leurs versions pour les comparer aux bases de données de vulnérabilités (CVE).

#### **Étapes Détaillées (à exécuter depuis srv-client) :**

1. **Installation et scan de base :**  
   sudo apt install nmap \-y  
   \# (Sur srv-main, assurez-vous qu'un service web est actif : sudo systemctl start nginx)

   \# Scan rapide des 1000 ports les plus courants.  
   nmap \<ip\_srv-main\>  
   \# Analyse : Vous devriez voir les ports 22 (ssh) et 80 (http) ouverts.

2. Scan de détection de services et de versions (-sV) :  
   Savoir qu'un port est ouvert est bien, savoir quoi écoute est mieux.  
   \# \-sV : Tente de déterminer la version du service  
   \# \-T4 : Accélère le scan (plus "bruyant" et détectable)  
   nmap \-sV \-T4 \<ip\_srv-main\>

   **Analyse :** La sortie sera plus riche : 80/tcp open http nginx 1.26.0. Cette version est cruciale pour rechercher des CVEs connues.  
3. **Scan Agressif et Détection d'OS (-A) :**  
   \# \-A active la détection de l'OS, la détection de version, le scan par script et le traceroute.  
   \# Nécessite les privilèges root pour certaines techniques.  
   sudo nmap \-A \<ip\_srv-main\>

   **Analyse :** Nmap tentera de deviner l'OS (Linux 5.15 \- 6.2) et lancera des scripts de découverte de base.  
4. Utilisation du Nmap Scripting Engine (NSE) :  
   Le NSE est la fonctionnalité la plus puissante de Nmap.  
   \# \--script vuln : Lance une collection de scripts qui recherchent les vulnérabilités les plus connues.  
   \# Ce scan peut être très long, intrusif et générer beaucoup de trafic.  
   sudo nmap \-sV \--script vuln \<ip\_srv-main\>  
