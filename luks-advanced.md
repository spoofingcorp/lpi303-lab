# **Chiffrement du Disque Système avec LUKS sur Ubuntu Server 24.04**

## **Introduction**

Ce document a pour objectif de fournir un guide technique exhaustif et une procédure de laboratoire ("lab") pour la mise en œuvre d'un chiffrement de disque système robuste sur Ubuntu Server 24.04 LTS. Il répond directement à la question de la faisabilité du chiffrement LUKS (Linux Unified Key Setup) lors de l'installation et détaille le processus de manière approfondie.  
La protection des données au repos ("data at rest") est une mesure de sécurité fondamentale dans l'écosystème informatique moderne. Le chiffrement complet du disque (Full Disk Encryption \- FDE) est la principale technologie permettant d'atteindre cet objectif, protégeant les informations sensibles contre l'accès physique non autorisé, que ce soit par le vol de matériel, une mise hors service inappropriée ou un accès interne malveillant.  
Pour y parvenir sous Linux, deux technologies fondamentales sont mises en synergie :

* **LUKS (Linux Unified Key Setup)** : Il s'agit du standard de facto pour le chiffrement de disque sous Linux. LUKS est une spécification qui définit un format sur disque indépendant de la plateforme pour les en-têtes de chiffrement, permettant une gestion aisée de plusieurs clés ou phrases de passe pour un même volume.

* **LVM (Logical Volume Manager)** : LVM fonctionne comme une couche d'abstraction de stockage. Son utilisation conjointe avec LUKS est une pratique courante et recommandée. LVM permet de créer et de gérer des "partitions" flexibles, appelées Volumes Logiques, à l'intérieur d'un unique conteneur chiffré par LUKS. Cette approche simplifie considérablement l'administration post-installation, en autorisant par exemple le redimensionnement des volumes sans affecter la couche de chiffrement sous-jacente.

Il est formellement confirmé que l'installateur d'Ubuntu Server 24.04, connu sous le nom de "Subiquity", supporte pleinement la configuration d'un système de fichiers chiffré avec LUKS et géré par LVM. Cependant, pour atteindre un niveau de contrôle granulaire et mettre en place l'architecture recommandée, il est impératif de suivre la procédure de partitionnement manuel ("Custom storage layout").
 

## **Partie 1 : Principes Fondamentaux de l'Architecture de Chiffrement**

### **1.1. Anatomie d'un Conteneur LUKS**

Un volume chiffré avec LUKS n'est pas simplement un ensemble de données brouillées ; il est structuré pour être à la fois sécurisé et gérable.

* **L'en-tête LUKS** : Situé au début du volume, cet en-tête est le centre de commande du chiffrement. Il contient des métadonnées cruciales, notamment les "slots de clés" (key slots), le sel cryptographique utilisé pour renforcer les clés, le nombre d'itérations de hachage (un facteur de résistance contre les attaques par force brute), et le type de chiffrement utilisé.

* **Clé Maîtresse et Slots de Clés** : LUKS emploie un mécanisme de chiffrement à deux niveaux. Une clé maîtresse (Master Key), générée aléatoirement lors de la création du volume, est utilisée pour chiffrer et déchiffrer l'intégralité des données du disque. Cette clé maîtresse est elle-même chiffrée à l'aide d'une phrase de passe fournie par l'utilisateur. Le résultat de ce chiffrement est stocké dans un des huit (ou plus) "slots" disponibles dans l'en-tête. Ce design ingénieux permet de changer une phrase de passe sans avoir à rechiffrer des téraoctets de données ; seule la clé maîtresse chiffrée dans le slot correspondant est modifiée. Il permet également d'autoriser plusieurs utilisateurs ou systèmes à déverrouiller le même volume, chacun avec sa propre phrase de passe.

* **LUKS1 vs LUKS2** : LUKS existe en deux versions principales. LUKS2 est le standard moderne, offrant des fonctionnalités améliorées telles qu'une plus grande résilience aux corruptions d'en-tête grâce à des sauvegardes de métadonnées et une meilleure gestion des algorithmes. Il est important de noter que certains composants plus anciens de la chaîne de démarrage, notamment le chargeur d'amorçage GRUB, peuvent avoir une compatibilité limitée avec LUKS2, ne supportant parfois que LUKS1. Cette limitation technique a des implications directes sur l'architecture de démarrage, comme nous le verrons plus loin.

### **1.2. La Synergie entre LUKS et LVM**

L'association de LUKS et LVM est la configuration standard pour les serveurs Linux nécessitant à la fois sécurité et flexibilité. L'empilement des couches technologiques se présente comme suit :

1. **Disque Physique** : Le matériel de stockage brut (SSD ou HDD).  
2. **Table de Partition (GPT)** : Le disque est divisé en partitions.  
3. **Conteneur Chiffré LUKS** : Une partition entière est dédiée à devenir un conteneur LUKS.  
4. **Volume Physique LVM (PV)** : Le conteneur LUKS déverrouillé est présenté au système comme un périphérique bloc, qui est ensuite initialisé comme un Volume Physique pour LVM.  
5. **Groupe de Volumes LVM (VG)** : Le PV est ajouté à un Groupe de Volumes, qui agit comme un pool de stockage.  
6. **Volumes Logiques LVM (LV)** : Le VG est découpé en Volumes Logiques, qui sont l'équivalent des partitions traditionnelles (/, /home, /var, etc.).  
7. **Systèmes de Fichiers** : Chaque LV est formaté avec un système de fichiers (ex: ext4).

Cette architecture offre des avantages pratiques considérables. Si le volume alloué à /var devient insuffisant, un administrateur peut le redimensionner en utilisant l'espace libre du VG, sans jamais avoir à interagir avec la couche de chiffrement sous-jacente. Cette flexibilité est cruciale dans un environnement de production.

### 

### **1.3. L'Architecture Cible Détaillée pour Ubuntu Server 24.04**

La procédure de laboratoire mettra en place la structure de partitionnement suivante, qui est un standard éprouvé pour les systèmes chiffrés :

1. **Partition 1 : EFI System Partition (ESP)**  
   * **Taille :** \~1 Go  
   * **Format :** FAT32  
   * **Point de montage :** /boot/efi  
   * **Chiffrement :** Non. Cette partition est indispensable pour le démarrage sur les systèmes UEFI modernes et doit être lisible par le firmware de la machine.

   

2. **Partition 2 : Partition de Démarrage**  
   * **Taille :** \~1 Go  
   * **Format :** ext4  
   * **Point de montage :** /boot  
   * **Chiffrement :** Non. Elle contient des éléments critiques pour le démarrage précoce, comme le noyau Linux (vmlinuz), l'Initial RAM File System (initramfs), et les fichiers de configuration de GRUB.

   

3. **Partition 3 : Partition Principale**  
   * **Taille :** Tout l'espace disque restant  
   * **Usage :** Contient le conteneur chiffré LUKS, qui sera ensuite utilisé comme Volume Physique (PV) pour LVM.

### **1.4. Analyse Approfondie : Pourquoi /boot Reste en Clair**

Laisser la partition /boot non chiffrée peut sembler contre-intuitif, mais c'est le résultat d'un compromis technique délibéré, dicté par la séquence de démarrage d'un système Linux.

1. **La Séquence de Démarrage (The Boot Chain)** : Le processus démarre lorsque le firmware UEFI de la carte mère exécute le chargeur d'amorçage GRUB, qu'il trouve sur la partition ESP (/boot/efi).  
2. **Chargement du Noyau** : GRUB, qui réside sur une partition non chiffrée, doit ensuite accéder à la partition /boot pour charger en mémoire le noyau Linux et l'initramfs. À ce stade très précoce, le système d'exploitation n'est pas encore en cours d'exécution, et seuls les pilotes les plus rudimentaires sont disponibles.  
3. **Limitations de GRUB** : Bien que GRUB possède une capacité limitée à déchiffrer des volumes LUKS (via la commande cryptomount), cette fonctionnalité est notoirement lente et principalement compatible avec l'ancien format LUKS1. Elle n'est pas conçue pour gérer les algorithmes et les fonctionnalités complexes de LUKS2, qui est le standard utilisé pour chiffrer la partition racine.  
4. **Le Rôle de l'initramfs** : Une fois le noyau et l'initramfs chargés en mémoire, l'initramfs prend le contrôle. C'est ce mini-système de fichiers qui contient les outils nécessaires (notamment cryptsetup) pour interagir avec l'utilisateur, demander la phrase de passe, et déverrouiller le conteneur LUKS principal.

5. **Montage de la Racine** : Ce n'est qu'après le déverrouillage réussi du conteneur que le système de fichiers racine (/) et tous les autres volumes logiques deviennent accessibles, permettant au processus de démarrage de se poursuivre normalement.

Ce processus révèle un choix d'ingénierie fondamental. Plutôt que de s'appuyer sur les capacités de déchiffrement limitées et fragiles de GRUB, les distributions Linux comme Ubuntu privilégient une approche plus robuste et universelle. Elles assurent la fiabilité et la performance du démarrage en laissant /boot en clair, garantissant ainsi la compatibilité avec les standards de chiffrement les plus récents (LUKS2) pour le reste du système.  
Cette architecture déplace la charge de la sécurité du démarrage vers d'autres mécanismes. Si /boot n'est pas protégé par la confidentialité (chiffrement), son intégrité doit être garantie. C'est là que des technologies comme **UEFI Secure Boot** deviennent non plus optionnelles, mais essentielles. Secure Boot utilise des signatures cryptographiques pour vérifier que le chargeur d'amorçage (GRUB) et, par extension, le noyau n'ont pas été altérés par un attaquant ayant un accès physique (une "evil maid attack"). La posture de sécurité globale ne repose donc pas sur une seule couche, mais sur un empilement de défenses : la sécurité physique du serveur, l'intégrité du démarrage garantie par Secure Boot, et la confidentialité des données assurée par LUKS.

## **Partie 2 : Laboratoire Pratique : Installation et Configuration Pas à Pas**

### **2.1. Prérequis et Préparation**

Avant de commencer, il est impératif de s'assurer que les éléments suivants sont en place :

* L'image ISO officielle d'Ubuntu Server 24.04 LTS (Noble Numbat) doit être téléchargée depuis le site officiel d'Ubuntu.  
* Un support d'installation USB amorçable doit être créé à l'aide d'un outil comme balenaEtcher, ou via la commande dd pour les utilisateurs avancés.  
* Le système cible doit être configuré dans le firmware pour démarrer en mode UEFI.  
* Si le disque cible contient une installation de Windows avec BitLocker, ce dernier doit être désactivé avant de procéder, car l'installateur Ubuntu ne peut pas redimensionner une partition chiffrée.  
* Une sauvegarde complète de toutes les données existantes sur le disque cible doit être effectuée. Le processus de partitionnement et d'installation est destructeur.

### **2.2. Lancement de l'Installateur Subiquity**

1. Démarrez l'ordinateur à partir du support USB créé.  
2. Naviguez dans les premiers écrans de l'installateur pour sélectionner la langue, la disposition du clavier et configurer la connexion réseau. Une connexion active est recommandée pour télécharger les mises à jour pendant l'installation.  
3. Lorsque le choix est proposé, sélectionnez "Interactive installation" pour suivre la procédure standard.

### 

### **2.3. Configuration Manuelle du Stockage : Le Cœur du Laboratoire**

C'est à l'étape "Storage configuration" que la mise en place du chiffrement s'effectue. L'installateur propose plusieurs options, dont le choix est déterminant.

| Option de Configuration | Description | Cas d'Usage | Contrôle |
| :---- | :---- | :---- | :---- |
| **Use an entire disk** | Efface le disque entier et installe Ubuntu avec un partitionnement par défaut (généralement / et swap). | Installation simple et rapide sur un disque vierge. | Minimal |
| **Use an entire disk (with LVM and encryption)** | Efface le disque entier, crée une configuration LVM et chiffre le tout avec LUKS. | Installation sécurisée et rapide, mais sans contrôle sur la taille des partitions. | Limité |
| **Custom storage layout** | Fournit un contrôle total sur la création des tables de partition, des partitions, des conteneurs chiffrés et des volumes LVM. | **Requis pour ce guide.** Permet de créer la structure ESP \+ /boot \+ LUKS/LVM. | Total |

Pour suivre ce guide, il est impératif de sélectionner **"Custom storage layout"**.

#### **Étape 1 : Création de la Partition Système EFI (ESP)**

1. Dans l'écran "Storage configuration", la liste des disques disponibles apparaît sous "AVAILABLE DEVICES".  
2. Sélectionnez le disque physique cible (ex: /dev/sda).  
3. Dans le menu qui s'affiche, choisissez l'option **"Use as Boot Device"**.  
4. L'installateur créera automatiquement une partition d'environ 1 Go, formatée en fat32 et assignée au point de montage /boot/efi.

#### **Étape 2 : Création de la Partition de Démarrage /boot**

1. Le disque affiche maintenant la partition ESP et de l'espace libre ("free space"). Sélectionnez cet espace libre.  
2. Choisissez l'option **"Add a GPT partition"**.  
3. Dans la boîte de dialogue, configurez les paramètres suivants :  
   * **Size :** 1G  
   * **Format :** ext4  
   * **Mount :** /boot  
4. Validez en sélectionnant **"\[Create\]"**.

#### **Étape 3 : Création de la Partition pour le Conteneur Chiffré**

1. Sélectionnez à nouveau l'espace libre restant sur le disque.  
2. Choisissez **"Add a GPT partition"**.  
3. Conservez la taille par défaut, qui correspond à tout l'espace restant.  
4. Pour le format, sélectionnez **"Leave unformatted"**. C'est une étape cruciale, car cette partition brute accueillera le conteneur LUKS.  
5. Validez avec **"\[Create\]"**.

#### **Étape 4 : Création du Groupe de Volumes LVM (chiffré)**

1. De retour sur l'écran principal, sous "AVAILABLE DEVICES", une nouvelle option est apparue : **"Create volume group (LVM)"**. Sélectionnez-la.  
2. Une boîte de dialogue s'ouvre pour configurer le groupe de volumes :  
   * **Name :** Donnez un nom significatif, par exemple vg\_ubuntu.  
   * **Devices :** Cochez la case correspondant à la grande partition non formatée créée à l'étape précédente.  
   * **Encrypt :** **Cochez cette case pour activer le chiffrement LUKS.**  
   * **Password / Passphrase :** Saisissez une phrase de passe forte et mémorisez-la. C'est la clé qui protège l'intégralité de vos données. Confirmez-la.  
3. Validez avec **"\[Create\]"**. L'installateur va maintenant formater la partition avec LUKS et initialiser un groupe de volumes LVM à l'intérieur.

#### **Étape 5 : Création des Volumes Logiques (LV)**

1. Le nouveau groupe de volumes (ex: vg\_ubuntu) apparaît maintenant dans la liste, avec de l'espace libre.  
2. Sélectionnez cet espace libre et choisissez **"Create Logical Volume"**.  
3. Répétez cette opération pour créer toutes les "partitions" nécessaires au système. Une configuration serveur typique est la suivante :  
   * **LV 1 (Racine)** : Name: lv\_root, Size: 30G (ou plus, selon les besoins), Format: ext4, Mount: /.  
   * **LV 2 (Swap)** : Name: lv\_swap, Size: 4G (ou 1 à 2 fois la RAM), Format: swap, Mount: (aucun).  
   * **LV 3 (Var)** : Name: lv\_var, Size: 20G (important pour les journaux, bases de données, et conteneurs Docker), Format: ext4, Mount: /var.  
   * **LV 4 (Home)** : Name: lv\_home, Size: laisser la taille par défaut pour utiliser tout l'espace restant, Format: ext4, Mount: /home.

#### **Étape 6 : Confirmation et Écriture sur le Disque**

1. Une fois tous les volumes logiques créés, l'écran de configuration du stockage affiche un résumé complet de l'architecture. Vérifiez attentivement que tout est correct.  
2. Sélectionnez **""** en bas de l'écran.  
3. Un écran de confirmation apparaît, avertissant que les modifications sont destructrices et vont être écrites sur le disque. Pour procéder, sélectionnez **"\[Continue\]"**.

### 

### **2.4. Finalisation de l'Installation**

L'installateur procède maintenant à la configuration du profil. Il vous sera demandé de définir le nom de votre machine, votre nom d'utilisateur et votre mot de passe de session. Il est également recommandé d'installer le serveur OpenSSH pour permettre une administration à distance. L'installation des paquets se poursuit jusqu'à la fin, après quoi un redémarrage est proposé.

## **Partie 3 : Gestion et Utilisation Post-Installation**

### **3.1. Le Premier Démarrage : L'Expérience Utilisateur**

Le premier démarrage d'un système chiffré est une étape critique.

* **L'invite de phrase de passe** : Après le menu de démarrage de GRUB, le processus de chargement du système s'interrompra. Une invite en mode texte apparaîtra sur la console : Please unlock disk \<disk\_name\>: (le nom du disque, par exemple dm-crypt-0, peut varier). À ce moment, l'utilisateur doit saisir la phrase de passe LUKS définie lors de l'installation et appuyer sur Entrée. Il est normal qu'aucun caractère (ni même des astérisques) ne s'affiche pendant la saisie.  
* **Dépannage des problèmes d'affichage de l'invite** : Dans certains environnements, notamment les machines virtuelles ou en raison de bogues spécifiques au noyau, l'invite de saisie de la phrase de passe peut ne pas s'afficher correctement, donnant l'impression que le système est bloqué. Ce problème survient souvent car l'invite est dirigée vers une console série non visible. La solution consiste à redémarrer, à éditer les options de démarrage dans GRUB (en appuyant sur 'e'), et à modifier la ligne commençant par linux pour soit supprimer les options quiet splash, soit ajouter console=tty1. Cette modification force l'affichage de l'invite sur la console texte principale, résolvant ainsi le problème. Pour rendre ce changement permanent, il faut modifier le fichier /etc/default/grub une fois le système démarré et exécuter sudo update-grub.  
* **Après le déverrouillage** : Une fois la phrase de passe correcte saisie, le conteneur LUKS est déverrouillé, les volumes LVM sont activés, et le reste du processus de démarrage se poursuit normalement jusqu'à l'invite de connexion de l'utilisateur.

### **3.2. Vérification et Audit de la Configuration**

Une fois la session ouverte, il est essentiel de vérifier que la configuration de chiffrement est correcte et active.

* Visualiser l'arborescence des périphériques blocs :  
  `lsblk -f`  
  La sortie de cette commande doit clairement montrer la partition physique avec le FSTYPE crypto\_LUKS, et en dessous, les volumes logiques mappés (ex: vg\_ubuntu-lv\_root) avec leurs systèmes de fichiers respectifs.  
* Obtenir le statut détaillé du volume chiffré :  
  `sudo cryptsetup status /dev/mapper/vg_ubuntu-lv_root`  
  Cette commande fournit des informations détaillées sur le chiffrement, y compris le cipher (ex: aes-xts-plain64), la taille de la clé, et le statut actif.  
* Vérifier la configuration LVM :  
  `sudo pvs && sudo vgs && sudo lvs`  
  Ces commandes permettent de lister respectivement les Volumes Physiques, les Groupes de Volumes et les Volumes Logiques, confirmant que la structure LVM est bien en place sur le périphérique déchiffré.

### **3.3. Administration des Clés LUKS avec cryptsetup**

La gestion des phrases de passe est une tâche d'administration courante.

* **Changer une phrase de passe existante** :  
  `sudo cryptsetup luksChangeKey /dev/sdXN`  
  (remplacer /dev/sdXN par la partition physique contenant le conteneur LUKS). Le système demandera d'abord l'ancienne phrase de passe avant de permettre la saisie de la nouvelle.  
* **Ajouter une nouvelle phrase de passe** : Il est possible d'avoir plusieurs phrases de passe pour déverrouiller le même volume, ce qui est utile pour accorder un accès à un autre administrateur ou pour créer une clé de secours.  
  `sudo cryptsetup luksAddKey /dev/sdXN`

* **Sauvegarde de l'en-tête LUKS** : C'est une opération de maintenance **critique**. Une corruption de l'en-tête LUKS, bien que rare, rendrait toutes les données définitivement irrécupérables. Il est donc impératif de le sauvegarder.  
  `sudo cryptsetup luksHeaderBackup /dev/sdXN --header-backup-file /chemin/vers/sauvegarde_header.img`

Ce fichier de sauvegarde doit être stocké sur un support externe, sécurisé et distinct du serveur.

## **Partie 4 : Considérations Avancées et Bonnes Pratiques**

### **4.1. Impact sur les Performances**

* **L'overhead du chiffrement** : Le chiffrement et le déchiffrement des données à la volée consomment des cycles CPU. Historiquement, cela pouvait entraîner une dégradation notable des performances des entrées/sorties (E/S).  
* **Le rôle d'AES-NI** : Les processeurs modernes (Intel et AMD) incluent un jeu d'instructions matérielles dédiées, appelées AES-NI (Advanced Encryption Standard New Instructions). Ces instructions accélèrent massivement les opérations de chiffrement et de déchiffrement AES. Sur un serveur moderne équipé d'un tel processeur, l'impact du chiffrement LUKS sur les performances des E/S est souvent négligeable, en particulier avec des SSD NVMe rapides dont le goulot d'étranglement n'est plus le CPU. La présence de ces instructions peut être vérifiée avec la commande grep aes /proc/cpuinfo.  
* **TRIM/Discard pour les SSD** : Pour maintenir les performances d'écriture des SSD sur le long terme, il est important que le système d'exploitation puisse informer le disque des blocs de données qui ne sont plus utilisés (une opération appelée TRIM). Pour permettre à cette commande de traverser la couche de chiffrement, l'option discard doit être ajoutée dans le fichier /etc/crypttab pour le volume concerné.

### **4.2. Stratégies de Sauvegarde et de Récupération**

Le chiffrement modifie la stratégie de sauvegarde.

* **Sauvegarde au niveau des fichiers** : C'est l'approche la plus simple et la plus courante. Des outils comme rsync, restic ou BorgBackup fonctionnent de manière transparente, car ils opèrent sur le système de fichiers une fois que celui-ci est déverrouillé et monté.  
* **Sauvegarde au niveau du bloc (images disque)** : Il est possible de créer une image du disque chiffré (par exemple avec dd). La sauvegarde résultante sera elle-même chiffrée. La restauration de cette image recréera le volume chiffré, qui nécessitera toujours la phrase de passe originale pour être déverrouillé.  
* **Plan de reprise après sinistre** : Un plan de récupération robuste doit inclure non seulement les sauvegardes des données, mais aussi un stockage sécurisé de la phrase de passe LUKS et du fichier de sauvegarde de l'en-tête. La perte de l'un ou l'autre de ces éléments peut entraîner une perte totale et irréversible des données.

### **4.3. Introduction au Déverrouillage par TPM 2.0**

Pour les serveurs qui nécessitent de redémarrer sans intervention manuelle (par exemple, après une mise à jour du noyau), le déverrouillage par phrase de passe peut être un obstacle.

* **Concept** : Un Trusted Platform Module (TPM) est une puce de sécurité matérielle présente sur la plupart des serveurs modernes. Elle peut stocker de manière sécurisée une clé de déchiffrement pour le volume LUKS. Au démarrage, le système peut demander au TPM de fournir cette clé pour déverrouiller automatiquement le disque, mais seulement si l'état du système (mesuré par les Platform Configuration Registers \- PCRs) n'a pas été modifié. Ces registres enregistrent des "empreintes" du firmware, du chargeur d'amorçage et du noyau. Si un de ces composants est modifié, le TPM refusera de libérer la clé, forçant une intervention manuelle.

* **Avantages et Inconvénients** : Le déverrouillage par TPM offre une grande commodité pour les redémarrages automatisés. Cependant, il introduit un compromis de sécurité : si un attaquant vole l'intégralité du serveur physique (disque et carte mère avec TPM), il pourrait potentiellement démarrer le système. La phrase de passe, en revanche, protège contre ce scénario. Le choix entre ces deux méthodes dépend donc du modèle de menace : le TPM protège contre les altérations logicielles malveillantes, tandis que la phrase de passe protège contre le vol physique du matériel.

## 

## **Partie 5 : Maintenance, Déplacement et Récupération de Données**

L'un des avantages majeurs de l'architecture LUKS sur LVM est sa portabilité. Un disque chiffré de cette manière est une entité autonome qui peut être déplacée entre des systèmes ou montée pour la récupération de données. Cette section détaille les procédures pour deux scénarios courants : la migration complète d'une VM et la récupération de données après un crash système.

### **5.1. Scénario 1 : Migration Complète du Disque vers une Nouvelle VM**

Le déplacement d'un disque système chiffré vers une nouvelle machine virtuelle (VM) est généralement simple.

1. **Détacher le disque :** Dans l'hyperviseur (VMware, KVM, VirtualBox, etc.), détachez le disque virtuel de la VM d'origine.

2. **Attacher le disque :** Attachez ce même disque virtuel à la nouvelle VM.

3. **Configuration du Démarrage :** Assurez-vous que la nouvelle VM est configurée pour démarrer à partir de ce disque.

4. **Démarrage :** Au démarrage, la nouvelle VM lira les partitions non chiffrées /boot/efi et /boot, chargera GRUB, puis le noyau et l'initramfs. L'initramfs détectera la partition racine chiffrée et présentera l'invite de phrase de passe exactement comme sur la machine d'origine. Une fois la phrase de passe correcte saisie, le système démarrera normalement.

Aucune reconfiguration du chiffrement n'est nécessaire car toutes les informations (en-tête LUKS, structure LVM) sont contenues sur le disque lui-même.

### **5.2. Scénario 2 : Montage pour Récupération de Données (Après Crash)**

Si le système d'exploitation d'origine est corrompu et ne démarre plus, mais que le disque est physiquement intact, vous pouvez monter les volumes chiffrés depuis un autre système Linux pour récupérer les fichiers.

**Prérequis :**

* Un environnement Linux fonctionnel. Cela peut être une autre VM ou un système démarré à partir d'une image "Live" d'Ubuntu.

* Les outils cryptsetup et lvm2 doivent être installés. Sur un système basé sur Debian/Ubuntu :  
  `sudo apt-get update`  
  `sudo apt-get install cryptsetup lvm2`

**Procédure de Montage :**

1. **Attacher le disque :** Attachez le disque chiffré à votre VM de récupération. Il apparaîtra comme un nouveau périphérique (par exemple, /dev/sdb).

2. **Identifier la partition LUKS :** Utilisez lsblk ou sudo fdisk \-l pour identifier la partition qui contient le conteneur LUKS. Elle sera la plus grande et n'aura pas de système de fichiers directement visible. Supposons qu'il s'agisse de /dev/sdb3.

3. **Déverrouiller le conteneur LUKS :** Utilisez la commande cryptsetup pour déverrouiller la partition. Vous devrez fournir la phrase de passe lorsque vous y serez invité.

`# Syntaxe : sudo cryptsetup luksOpen /dev/partition_chiffrée nom_mappage`  
`sudo cryptsetup luksOpen /dev/sdb3 ubuntu_crypted`  
Cette commande crée un périphérique mappé déchiffré à /dev/mapper/ubuntu\_crypted.

4. **Activer le Groupe de Volumes LVM :** Le périphérique déchiffré est un Volume Physique LVM. Vous devez scanner et activer le Groupe de Volumes qu'il contient pour rendre les Volumes Logiques accessibles.  
   `# Scanner les disques pour trouver les groupes de volumes`  
   `sudo vgscan`

   `# Activer tous les groupes de volumes trouvés`  
   `sudo vgchange -ay`

Si vous rencontrez un conflit de noms avec un groupe de volumes existant sur le système de récupération, vous devrez peut-être renommer le VG en utilisant son UUID avec vgrename.

5. **Identifier et Monter les Volumes Logiques :** Une fois le VG activé, les LV apparaîtront dans /dev/mapper/. Vous pouvez les lister avec sudo lvs ou ls /dev/mapper/.  
   `# Créer un point de montage`  
   `sudo mkdir /mnt/recovery`

   `# Monter le volume logique racine`  
   `# Le nom sera typiquement <nom_vg>-<nom_lv>, ex: vg_ubuntu-lv_root`  
   `sudo mount /dev/mapper/vg_ubuntu-lv_root /mnt/recovery`

   `# Si nécessaire, monter d'autres volumes comme /home`  
   `sudo mkdir /mnt/recovery/home`  
   `sudo mount /dev/mapper/vg_ubuntu-lv_home /mnt/recovery/home`

6. **Accéder aux données :** Vos fichiers sont maintenant accessibles dans /mnt/recovery. Vous pouvez les copier sur un autre support.

7. **Nettoyage (Démontage et Verrouillage) :** Une fois la récupération terminée, il est crucial de démonter proprement les volumes et de refermer le conteneur chiffré.  
   `# Démontage des volumes (en ordre inverse du montage)`  
   `sudo umount /mnt/recovery/home`  
   `sudo umount /mnt/recovery`

   `# Désactiver le groupe de volumes`  
   `sudo vgchange -an vg_ubuntu`

   `# Verrouiller le conteneur LUKS`  
   `sudo cryptsetup luksClose ubuntu_crypted`

Cette procédure garantit un accès sécurisé aux données même en cas de défaillance complète du système d'exploitation, démontrant la robustesse et la flexibilité de la solution de chiffrement LUKS sur LVM.

## **Conclusion**

Ce guide a démontré de manière exhaustive qu'il est non seulement possible, mais également recommandé, de configurer un serveur Ubuntu 24.04 avec un chiffrement de disque système robuste et flexible. En utilisant la méthode standard de l'industrie, qui combine LUKS et LVM, les administrateurs peuvent atteindre un haut niveau de sécurité pour les données au repos.

Les points clés à retenir sont les suivants :

* L'utilisation de l'option de partitionnement manuel **"Custom storage layout"** dans l'installateur Subiquity est indispensable pour mettre en œuvre l'architecture de chiffrement recommandée.  
* La présence d'une partition /boot non chiffrée est un choix technique délibéré qui garantit la fiabilité du démarrage. La sécurité de cette configuration doit être renforcée par des mesures compensatoires comme UEFI Secure Boot pour garantir l'intégrité de la chaîne de démarrage.  
* La gestion post-installation est aussi cruciale que la configuration initiale. La maîtrise des commandes cryptsetup, la gestion rigoureuse des phrases de passe et, surtout, la sauvegarde systématique de l'en-tête LUKS sont des pratiques non négociables.

En définitive, le chiffrement de disque ne doit pas être considéré comme une solution isolée, mais comme la première ligne de défense fondamentale pour les données au repos. Il constitue la base sur laquelle une posture de sécurité multicouche peut être construite, protégeant ainsi les actifs informationnels les plus précieux de l'organisation.

#### **Sources des citations**

1\. How to install Ubuntu with LUKS Encryption on LVM \- GitHub Gist, https://gist.github.com/superjamie/d56d8bc3c9261ad603194726e3fef50f 2\. Configuring storage \- Ubuntu installation documentation, https://canonical-subiquity.readthedocs-hosted.com/en/latest/howto/configure-storage.html 3\. Installing Ubuntu 24.04 Server with LVM and Encryption Study ..., https://quizlet.com/study-guides/installing-ubuntu-24-04-server-with-lvm-and-encryption-b0aab65c-7e91-4509-9d70-e98d807a66ec 4\. Full disk encryption, including /boot: Unlocking LUKS devices from ..., https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html 5\. Setting up LUKS encryption on an existing Ubuntu partition | Karthik ..., https://karthikkaranth.me/blog/setting-up-luks-encryption-on-an-existing-ubuntu-partition/ 6\. Should you encrypt your boot partition? : r/archlinux \- Reddit, https://www.reddit.com/r/archlinux/comments/1fhakww/should\_you\_encrypt\_your\_boot\_partition/ 7\. Can I encrypt my boot partition with LUKS encryption? \- Red Hat Customer Portal, https://access.redhat.com/solutions/3419031 8\. Install Ubuntu Desktop, https://ubuntu.com/tutorials/install-ubuntu-desktop 9\. Full Disk Encryption: Install Ubuntu 24.04 Securely, https://www.tecmint.com/encrypt-ubuntu-24-04-installation/ 10\. How to Dual-Boot Ubuntu 24.04 \- 24.10 and Windows (10 or 11\) with Encryption, https://www.mikekasberg.com/blog/2024/05/20/dual-boot-ubuntu-24-04-and-windows-with-encryption.html 11\. Ubuntu Server 24.04: Installation, Partitioning, and Network Configuration \- WellWells, https://wellstsai.com/en/post/ubuntu-setup/ 12\. Ubuntu desktop LUKS install stuck at boot splash screen · Issue \#3555 · utmapp/UTM, https://github.com/utmapp/UTM/issues/3555 13\. Cannot enter luks password at boot after kernel 6.8.0-50 upgrade \- Launchpad Bugs, https://bugs.launchpad.net/bugs/2091864 14\. \[RESPONDED\] Any issues with LUKS or full disk encryption in general? \- Linux, https://community.frame.work/t/responded-any-issues-with-luks-or-full-disk-encryption-in-general/10725 15\. Ubuntu 24.04 installer not showing encrypted lvm partitions (luks), https://askubuntu.com/questions/1511577/ubuntu-24-04-installer-not-showing-encrypted-lvm-partitions-luks 16\. Ubuntu 24.04, Luks2, TPM2 auto unlock \- Reddit, https://www.reddit.com/r/Ubuntu/comments/1djvp00/ubuntu\_2404\_luks2\_tpm2\_auto\_unlock/ 17\. Mount encrypted volumes from command line? \- Ask Ubuntu, https://askubuntu.com/questions/63594/mount-encrypted-volumes-from-command-line 18\. Encrypt Drives Using LUKS on Oracle Linux, https://docs.oracle.com/en/learn/ol-luks/ 19\. Recover data from ubuntu encrypted drive (LUKS) \- Server Fault, https://serverfault.com/questions/580195/recover-data-from-ubuntu-encrypted-drive-luks 20\. How To MOUNT A LUKS Encrypted Linux Filesystem Unlocked Partition VIA USB on another LINUX OS to retrieve files, https://unix.stackexchange.com/questions/697054/how-to-mount-a-luks-encrypted-linux-filesystem-unlocked-partition-via-usb-on-ano