# **(AIDE) \- Advanced Intrusion Detection Environment**

## **1\. Introduction à la Surveillance de l'Intégrité des Fichiers avec AIDE**

La sécurisation d'un système d'information repose sur une stratégie de défense en profondeur, où de multiples couches de contrôle collaborent pour protéger les actifs numériques. Parmi ces contrôles, la surveillance de l'intégrité des fichiers (File Integrity Monitoring \- FIM) joue un rôle fondamental. Elle agit comme un gardien vigilant, capable de détecter des modifications non autorisées sur des fichiers critiques du système, qui sont souvent les premiers signes d'une compromission réussie. L'Advanced Intrusion Detection Environment (AIDE) est une solution open-source de premier plan dans ce domaine, fournissant un mécanisme robuste et fiable pour assurer cette surveillance.

### **1.1. Principes Fondamentaux d'un HIDS (Host-Based Intrusion Detection System)**

AIDE appartient à la catégorie des systèmes de détection d'intrusion basés sur l'hôte (Host-Based Intrusion Detection System \- HIDS). Contrairement aux systèmes de détection d'intrusion basés sur le réseau (NIDS) qui analysent le trafic réseau, un HIDS se concentre sur les activités internes d'un système informatique spécifique. Il examine les journaux système, les processus en cours d'exécution et, dans le cas d'AIDE, l'état du système de fichiers pour identifier des anomalies ou des activités malveillantes.

Il est essentiel de bien délimiter le périmètre fonctionnel d'AIDE. Sa spécialité est la surveillance de l'intégrité des fichiers (FIM). Il ne se charge pas de l'analyse en temps réel des journaux système ni de la recherche de rootkits, des tâches qui sont plutôt dévolues à d'autres HIDS plus complets comme OSSEC ou Splunk. AIDE a été initialement développé comme une alternative libre et gratuite au logiciel commercial Tripwire, partageant avec lui la même philosophie de base : la détection de changements par rapport à un état de référence connu.

Cette spécialisation fait d'AIDE un outil extrêmement performant pour sa mission principale. En se concentrant exclusivement sur l'intégrité des fichiers, il offre une granularité et une précision inégalées pour détecter des modifications qui pourraient échapper à d'autres types de systèmes de sécurité.

### 

### **1.2. Le Mécanisme d'AIDE : Base de Référence, Instantanés et Vérification Cryptographique**

Le fonctionnement d'AIDE repose sur un concept simple mais puissant : la création d'un "instantané" (snapshot) du système de fichiers à un instant T, qui sert de base de référence ("baseline") pour toutes les comparaisons futures. Ce processus se déroule en trois étapes distinctes :
  
1. **Création de l'Instantané** : Lors de son initialisation, AIDE parcourt le système de fichiers en suivant un ensemble de règles définies dans son fichier de configuration, /etc/aide.conf. Ces règles, basées sur des expressions régulières, lui indiquent précisément quels fichiers et répertoires inclure ou exclure de la surveillance.

2. **Génération de la Base de Données** : Pour chaque fichier sélectionné, AIDE collecte une série d'attributs. Ces attributs peuvent inclure les permissions, le propriétaire, le groupe, les horodatages (modification, accès, changement de statut), le numéro d'inode, et surtout, une ou plusieurs empreintes cryptographiques (hashes). AIDE supporte une large gamme d'algorithmes de hachage, tels que MD5, SHA1, SHA256, SHA512, RMD160, Tiger, et bien d'autres, ce qui garantit un très haut niveau d'assurance quant à l'intégrité du contenu des fichiers. Toutes ces informations sont compilées dans une base de données, généralement stockée sous forme d'un fichier compressé.  

3. **Vérification de l'Intégrité** : Lors des exécutions ultérieures, AIDE répète le processus de scan du système de fichiers mais, cette fois, il compare les attributs et les empreintes des fichiers actuels avec ceux enregistrés dans la base de référence. Toute divergence, qu'il s'agisse d'un fichier ajouté, supprimé ou modifié, est méticuleusement enregistrée dans un rapport détaillé.

Ce mécanisme fait de la base de données AIDE le pivot central de la confiance. Si cette base est intègre et a été générée sur un système sain, alors les rapports de AIDE sont une source fiable d'information sur l'état du système. Cependant, si un attaquant parvient à modifier cette base de données, il peut y inscrire ses propres modifications malveillantes, les faisant ainsi apparaître comme légitimes lors des vérifications futures. Cette réalité implique que la sécurisation de la base de données AIDE elle-même est une composante non négociable de son déploiement.

### 

### **1.3. Importance Stratégique d'AIDE : Conformité, Analyse Forensique et Renforcement des Systèmes**

L'utilité d'AIDE s'étend bien au-delà de la simple détection d'intrusion. Il s'agit d'un outil polyvalent qui s'intègre dans plusieurs facettes de la gestion de la sécurité des systèmes.

* **Renforcement des Systèmes (Server Hardening)** : Après avoir appliqué des configurations de sécurité strictes sur un serveur, AIDE permet de vérifier que ces configurations restent en place. Il agit comme une alarme qui se déclenche si un fichier de configuration critique est modifié, si un service est activé de manière inattendue ou si un logiciel non autorisé est installé.

* **Conformité Réglementaire** : De nombreuses normes et réglementations de sécurité, telles que PCI-DSS (pour les paiements par carte bancaire), HIPAA (pour les données de santé) et ISO/IEC 27001, exigent explicitement la mise en place de mécanismes de surveillance de l'intégrité des fichiers. Le déploiement d'AIDE est une méthode directe et reconnue pour satisfaire à ces exigences de conformité.

* **Analyse Forensique (Forensic Investigations)** : En cas d'incident de sécurité, le temps est un facteur critique. Les rapports d'AIDE fournissent une chronologie précise des modifications apportées au système de fichiers. Les enquêteurs peuvent utiliser la base de référence comme un témoin historique pour déterminer exactement quels fichiers ont été modifiés, quand la modification a eu lieu, et potentiellement comment l'intrusion s'est déroulée. Cela permet de circonscrire rapidement l'étendue de la compromission et d'identifier les portes dérobées (backdoors) ou les rootkits installés par l'attaquant.

* **Gestion des Changements et Audits** : AIDE est également un excellent outil pour la gestion des changements. Après une fenêtre de maintenance ou une mise à jour logicielle, l'exécution d'une vérification AIDE permet de s'assurer que seules les modifications prévues ont été effectuées. Tout changement inattendu peut être immédiatement identifié et investigué, réduisant ainsi le risque d'erreurs de configuration ou de modifications involontaires.

En somme, AIDE est un contrôle de détection. Il n'empêche pas une attaque de se produire, mais il fournit les preuves irréfutables qu'un changement a eu lieu. Son rôle est de réduire le "temps de séjour" d'un attaquant sur un système en alertant rapidement les administrateurs de sa présence, transformant une intrusion furtive en un événement de sécurité visible et actionnable.

## 

## **2\. Installation et Déploiement sur Ubuntu 24.04**

Le déploiement d'AIDE sur un système Ubuntu 24.04 LTS est un processus structuré. Il commence par la préparation du système et l'installation du paquet logiciel, suivi par la vérification de son installation et la familiarisation avec son arborescence de fichiers.

### **2.1. Prérequis et Préparation du Système**

Avant de procéder à l'installation, il est impératif de s'assurer que le système est dans un état optimal et que les prérequis sont satisfaits.

* **Système d'exploitation** : Le serveur doit fonctionner sous Ubuntu 24.04 LTS.  
* **Privilèges** : L'utilisateur doit disposer de privilèges sudo pour installer des paquets et exécuter les commandes d'administration d'AIDE.  
* **Mise à jour du système** : Il est fortement recommandé de mettre à jour le système avant toute nouvelle installation. Cela garantit que toutes les dépendances sont à jour et que les derniers correctifs de sécurité sont appliqués. Cette opération s'effectue avec la commande suivante :

`sudo apt update && sudo apt upgrade -y`

Cette étape préventive minimise les risques de conflits de paquets et assure que l'environnement est stable.

### 

### **2.2. Procédure d'Installation Détaillée du Paquet AIDE**

Le paquet AIDE est disponible dans les dépôts officiels d'Ubuntu, ce qui rend son installation simple et directe.

1. **Installation du paquet** : Utilisez le gestionnaire de paquets apt pour installer AIDE :  
   `sudo apt install aide`

2. **Gestion de l'invite de configuration de Postfix** : Au cours de l'installation, il est très probable qu'une boîte de dialogue de configuration pour Postfix apparaisse. Postfix est un agent de transfert de courrier (MTA) qu'AIDE peut utiliser pour envoyer ses rapports par courriel. Il est important de comprendre que Postfix n'est pas une dépendance stricte pour le fonctionnement d'AIDE.

   * Si vous n'avez pas l'intention de configurer immédiatement les rapports par courriel ou si un autre MTA est déjà en place, vous pouvez choisir l'option **"No configuration"**.  
   * Si vous prévoyez d'envoyer des courriels directement depuis ce serveur, l'option **"Internet Site"** est un bon choix par défaut.  
   * Cette configuration peut être modifiée ultérieurement. Pour ce laboratoire, choisir "No configuration" est une option sûre pour se concentrer d'abord sur le fonctionnement de base d'AIDE.

### **2.3. Vérification de l'Installation et Exploration de l'Arborescence AIDE**

Une fois l'installation terminée, il est crucial de vérifier qu'elle s'est déroulée correctement et de se familiariser avec les fichiers et répertoires créés.

1. **Vérification de la version** : Exécutez la commande suivante pour afficher la version d'AIDE installée ainsi que les options avec lesquelles il a été compilé.  
   `aide --version`  
   ou  
   `aide -v`

La sortie de cette commande confirme non seulement que le binaire est exécutable, mais elle listera également les fonctionnalités supportées, telles que WITH\_POSIX\_ACL, WITH\_SELINUX, WITH\_XATTR, etc. C'est une information utile pour savoir quels types d'attributs de fichiers AIDE est capable de surveiller.

2. **Exploration de l'arborescence** : L'installation d'AIDE crée une structure de répertoires standard qu'il est important de connaître :

   * /etc/aide/ : Ce répertoire contient les fichiers de configuration. Le fichier principal est aide.conf. Dans les installations modernes d'Ubuntu, il peut également contenir un sous-répertoire aide.conf.d pour les fichiers de configuration additionnels.

   * /var/lib/aide/ : C'est l'emplacement par défaut pour les bases de données d'AIDE. Vous y trouverez aide.db.gz (la base de référence active) et aide.db.new.gz (la nouvelle base générée lors d'une initialisation ou d'une mise à jour).

   * /etc/cron.daily/aide : Le paquet AIDE installe par défaut un script cron qui exécute une vérification quotidienne. Cela signifie qu'une fois la base de données initialisée, AIDE commencera automatiquement à surveiller le système et à générer des rapports.

   

La présence de ce script cron par défaut est une caractéristique importante des paquets Debian/Ubuntu. Elle implique que le système est conçu pour une surveillance active dès le départ. Par conséquent, il est primordial de configurer AIDE de manière appropriée *avant* d'initialiser la base de données pour la première fois. Si l'initialisation est faite avec la configuration par défaut, les rapports quotidiens risquent d'être inondés de "faux positifs" provenant de fichiers volatils (journaux, caches, etc.), rendant les alertes difficiles à gérer. La séquence correcte est donc : configurer, puis initialiser.

Enfin, il est à noter que les guides plus anciens mentionnent souvent une commande update-aide.conf. Cette commande, qui servait à fusionner les fichiers de configuration du répertoire aide.conf.d, est obsolète depuis la version 0.17 d'AIDE et n'est plus présente sur Ubuntu 24.04. Les versions modernes d'AIDE traitent directement les fichiers de ce répertoire grâce à des directives d'inclusion, simplifiant le processus de configuration.

## 

## **3\. Configuration Avancée de aide.conf**

Le fichier /etc/aide.conf est le cerveau d'AIDE. C'est ici que l'on définit avec une grande précision le périmètre de la surveillance. Une configuration bien pensée est la clé d'un déploiement réussi, permettant de trouver le juste équilibre entre une sécurité exhaustive et la gestion du "bruit" généré par les changements légitimes du système.

### **3.1. Anatomie du Fichier de Configuration : Directives, Macros et Règles de Sélection**

Le fichier aide.conf est structuré autour de trois types de lignes, chacune ayant un rôle spécifique :

* **Lignes de configuration (directives)** : Elles définissent les paramètres globaux de fonctionnement d'AIDE. Leur format est parametre=valeur. Les directives les plus importantes incluent :

  * database\_in=file:/var/lib/aide/aide.db.gz : Spécifie l'emplacement de la base de données de référence à utiliser pour la comparaison. La directive plus ancienne database est dépréciée et doit être remplacée par database\_in.

  * database\_out=file:/var/lib/aide/aide.db.new.gz : Indique où écrire la nouvelle base de données lors d'une initialisation (--init) ou d'une mise à jour (--update).

  * report\_url=file:/var/log/aide/aide.log : Définit la destination des rapports. Plusieurs URL peuvent être spécifiées. Les options courantes sont stdout (sortie standard), stderr (sortie d'erreur), file:/chemin/vers/fichier et syslog:FACILITY (pour envoyer les rapports au service syslog).

  * log\_level et report\_level : Permettent de contrôler le niveau de verbosité des journaux et des rapports, allant de minimal à list\_entries et changed\_attributes.


* **Lignes de macro** : Elles permettent de définir des variables pour simplifier la configuration. La syntaxe est @@define VARIABLE valeur. Par exemple, @@define DBDIR /var/lib/aide permet d'utiliser ${DBDIR} dans le reste du fichier.

* **Lignes de sélection (règles)** : Ce sont les lignes les plus nombreuses et les plus importantes. Elles utilisent des expressions régulières pour spécifier quels fichiers et répertoires surveiller, et quel niveau de surveillance leur appliquer. Elles commencent par un chemin (/etc), suivi d'un groupe d'attributs (NORMAL). Les lignes commençant par \! sont des règles d'exclusion.

### 

### **3.2. Définition des Règles de Surveillance Personnalisées : Analyse des Attributs**

Plutôt que d'appliquer des règles génériques, la véritable puissance d'AIDE réside dans la capacité à définir des "groupes d'attributs" personnalisés. Un groupe est simplement un nom associé à une combinaison d'attributs à vérifier. Par exemple, MyRule \= p+u+g+sha512 définit une règle nommée MyRule qui vérifiera les permissions, l'utilisateur, le groupe et l'empreinte SHA-512 d'un fichier.

Le choix des attributs à surveiller est une décision stratégique. Surveiller trop d'attributs (comme l'heure d'accès, atime) sur des fichiers qui changent constamment générera un volume d'alertes ingérable. Ne pas en surveiller assez (omettre les permissions, par exemple) pourrait laisser passer des modifications critiques. Le tableau suivant détaille les attributs les plus importants et leurs cas d'usage.

**Table 1: Référence Complète des Attributs de Surveillance AIDE**

| Code d'Attribut | Description | Cas d'Usage et Recommandations |
| :---- | :---- | :---- |
| p | Permissions | Essentiel. Détecte les modifications de permissions (ex: chmod 777 ou chmod \+s). À utiliser sur tous les fichiers critiques. |
| i | Numéro d'Inode | Utile pour détecter le remplacement d'un fichier (suppression puis recréation), même si le nom est identique. |
| n | Nombre de liens (hard links) | Spécifique, utile pour surveiller les fichiers système qui utilisent des liens durs. |
| u | Propriétaire (UID) | Essentiel. Détecte les changements de propriétaire (ex: chown root). |
| g | Groupe (GID) | Essentiel. Détecte les changements de groupe propriétaire. |
| s | Taille du fichier | Monitore la taille exacte du fichier. Ne pas utiliser sur les fichiers journaux ou les fichiers qui croissent légitimement. |
| b | Nombre de blocs | Similaire à la taille, mais au niveau du système de fichiers. |
| m | mtime (Modification time) | Heure de la dernière modification du contenu du fichier. Fondamental pour la détection de modification de contenu. |
| c | ctime (Change time) | Heure du dernier changement de statut du fichier (contenu, permissions, propriétaire, etc.). Plus sensible que mtime. |
| a | atime (Access time) | Heure du dernier accès. **À éviter sur la plupart des systèmes en production**, car la simple lecture d'un fichier déclenche une alerte. |
| S | Taille croissante | Règle spéciale pour les fichiers journaux. N'alerte que si la taille du fichier diminue, ce qui peut indiquer une altération ou une rotation anormale. |
| H | Tous les hachages compilés | Un raccourci pour vérifier tous les algorithmes de hachage disponibles. |
| md5, sha1, sha256, sha512, rmd160, tiger | Hachages spécifiques | **Le plus important pour l'intégrité du contenu.** sha256 ou sha512 sont recommandés pour une sécurité forte. |
| acl | Listes de Contrôle d'Accès (ACLs) | À utiliser si votre système utilise des ACLs POSIX pour une gestion fine des permissions. |
| xattrs | Attributs Étendus | Utile pour les systèmes de fichiers qui utilisent des attributs étendus pour stocker des métadonnées. |
| selinux | Contexte SELinux | Indispensable sur les systèmes utilisant SELinux (plus courant sur les distributions basées sur Red Hat, mais disponible sur Ubuntu). |

### 

### **3.3. Élaboration d'une Configuration Pratique pour un Serveur de Production**

Un fichier aide.conf efficace pour un serveur de production doit surveiller de manière agressive les zones statiques et critiques, tout en excluant intelligemment les zones dynamiques et volatiles pour minimiser les faux positifs.

**Cibles de Surveillance Essentielles :**

* **Binaires système (/bin, /sbin, /usr/bin, /usr/sbin)** : Pour détecter l'installation de logiciels malveillants ou le remplacement de commandes légitimes par des versions compromises (chevaux de Troie).

* **Bibliothèques système (/lib\*, /usr/lib\*)** : Pour détecter l'altération de bibliothèques partagées, une technique d'attaque avancée.

* **Fichiers de configuration (/etc)** : Le cœur de la configuration du système. Toute modification non autorisée ici peut avoir des conséquences graves sur la sécurité.

* **Noyau et chargeur d'amorçage (/boot)** : Pour détecter les bootkits ou les modifications du noyau.

**Stratégies d'Exclusion Intelligente :**

L'utilisation de la directive d'exclusion (\!) est fondamentale pour rendre AIDE gérable. Les répertoires suivants sont des candidats typiques à l'exclusion sur un serveur Ubuntu :

* \!/proc et \!/sys : Systèmes de fichiers virtuels du noyau.  
* \!/dev : Fichiers de périphériques, très dynamiques.  
* \!/run : Fichiers d'exécution et sockets.  
* \!/tmp et \!/var/tmp : Répertoires de fichiers temporaires.  
* \!/var/log : Les fichiers journaux doivent être surveillés avec une règle spéciale (S) ou exclus s'ils sont gérés par un système de journalisation centralisé.  
* \!/var/cache : Caches d'applications.  
* \!/var/spool : Files d'attente (impression, courrier, etc.).

**Exemple de Fichier aide.conf Complet et Annoté pour Ubuntu 24.04 :**  
Voici un exemple de configuration robuste qui peut servir de point de départ pour un serveur de production standard.  
```
`# #############################################################################`  
`# Fichier de configuration AIDE pour un serveur Ubuntu 24.04 LTS`  
`# #############################################################################`

`# =============================================================================`  
`# 1. DIRECTIVES GLOBALES`  
`# =============================================================================`  
`# Définition des emplacements des bases de données.`  
`# 'database_in' est la base de référence pour la comparaison.`  
`# 'database_out' est la destination pour la nouvelle base lors d'une mise à jour.`  
`database_in=file:/var/lib/aide/aide.db.gz`  
`database_out=file:/var/lib/aide/aide.db.new.gz`

`# Définition de la destination des rapports.`  
`# Nous envoyons les rapports vers un fichier log et vers la sortie standard.`  
`report_url=file:/var/log/aide/aide.log`  
`report_url=stdout`

`# Configuration de la verbosité. 'changed_attributes' fournit des détails utiles.`  
`log_level=warning`  
`report_level=changed_attributes`

`# =============================================================================`  
`# 2. DÉFINITION DES GROUPES D'ATTRIBUTS PERSONNALISÉS`  
`# =============================================================================`  
`# Règle pour les répertoires : surveille les métadonnées, pas le contenu.`  
`DIR = p+i+n+u+g`

`# Règle pour les fichiers de configuration : métadonnées + contenu (SHA512).`  
`CONFIG = p+i+n+u+g+s+m+c+sha512`

`# Règle pour les binaires et bibliothèques statiques : comme CONFIG, mais sans la taille.`  
`# La taille peut changer lors des mises à jour, mais le contenu (hash) ne devrait pas`  
`# changer en dehors d'une mise à jour légitime.`  
`STATIC = p+i+n+u+g+m+c+sha512`

`# Règle pour les fichiers journaux : métadonnées + vérification de taille croissante.`  
`LOG = p+i+n+u+g+S`

`# Règle de surveillance complète (pour des cas spécifiques, très verbeux).`  
`FULL = ALL-a-I`

`# =============================================================================`  
`# 3. RÈGLES DE SÉLECTION ET D'EXCLUSION`  
`# =============================================================================`

`# -----------------------------------------------------------------------------`  
`# Exclusions globales pour réduire le bruit`  
`# -----------------------------------------------------------------------------`  
`# Systèmes de fichiers virtuels et d'exécution`  
`!/proc`  
`!/sys`  
`!/dev`  
`!/run`  
`# Fichiers temporaires, caches et spools`  
`!/tmp`  
`!/var/tmp`  
`!/var/cache`  
`!/var/spool`  
`!/var/lib/apt/lists/.*`  
`# Exclure les logs par défaut (ils seront gérés spécifiquement si besoin)`  
`!/var/log/.*`

`# -----------------------------------------------------------------------------`  
`# Surveillance des zones critiques du système`  
`# -----------------------------------------------------------------------------`  
`# Noyau et chargeur d'amorçage`  
`/boot   STATIC`

`# Binaires et bibliothèques`  
`/bin    STATIC`  
`/sbin   STATIC`  
`/lib    STATIC`  
`/lib64  STATIC`  
`/usr/bin    STATIC`  
`/usr/sbin   STATIC`  
`/usr/lib    STATIC`  
`/usr/lib64  STATIC`  
`/usr/local/bin  STATIC`  
`/usr/local/sbin STATIC`  
`/usr/local/lib  STATIC`

`# Fichiers de configuration système`  
`/etc    CONFIG`

`# -----------------------------------------------------------------------------`  
`# Surveillance des données utilisateur et des applications (à adapter)`  
`# -----------------------------------------------------------------------------`  
`# Répertoires home (surveiller les fichiers de configuration cachés)`  
`/home/[^/]+/\.bashrc   CONFIG`  
`/home/[^/]+/\.profile  CONFIG`  
`/home/[^/]+/\.ssh      CONFIG`

`# Répertoire racine de l'utilisateur root`  
`/root   CONFIG`

`# Exemple pour un serveur web (surveiller les fichiers statiques)`  
`#!/var/www/html/uploads # Exclure les uploads utilisateurs`  
`# /var/www/html         STATIC`

`# -----------------------------------------------------------------------------`  
`# Surveillance spécifique des logs (si nécessaire)`  
`# -----------------------------------------------------------------------------`  
`# Surveiller le journal d'authentification pour détecter les effacements`  
`# /var/log/auth.log     LOG`
```

Cette configuration est un point de départ solide. Chaque administrateur devra l'adapter à son environnement spécifique, en ajoutant des règles pour les applications critiques et en affinant les exclusions pour correspondre à l'activité normale de son serveur.

## **4\. Gestion de la Base de Données et Opérations Fondamentales**

La maîtrise des commandes de base d'AIDE est essentielle pour son administration quotidienne. Le cycle de vie de la base de données – initialisation, vérification et mise à jour – constitue le cœur des opérations de surveillance.

### **4.1. Création de la Base de Référence Initiale (aideinit vs. aide \--init)**

La première étape, après avoir configuré aide.conf, est de créer la base de données de référence. C'est cet "instantané" initial qui servira de mètre étalon pour toutes les futures vérifications.

Sur les systèmes basés sur Debian comme Ubuntu, deux commandes semblent permettre d'atteindre cet objectif : aideinit et aide \--init. Il est crucial de comprendre leur différence. aide \--init est l'appel direct au binaire AIDE pour effectuer une initialisation. Cependant, cette commande peut nécessiter de spécifier manuellement le chemin vers le fichier de configuration (-c /etc/aide/aide.conf).

La commande recommandée sur Ubuntu est aideinit. Il s'agit d'un script wrapper fourni par le paquet Ubuntu qui se charge d'appeler aide avec les bons paramètres et chemins par défaut pour la distribution. Utiliser aideinit garantit la cohérence et simplifie la procédure.

1. **Exécutez la commande d'initialisation** :  
   `sudo aideinit`

2. **Analysez la sortie** : Le processus va scanner l'ensemble du système de fichiers selon les règles de aide.conf, calculer les attributs et les hachages, et enfin créer une nouvelle base de données. Cette opération peut être longue et consommer des ressources CPU et I/O importantes, en particulier sur des systèmes avec de nombreux fichiers. La sortie indiquera la création du fichier /var/lib/aide/aide.db.new.gz.

### **4.2. Activation de la Base de Données pour la Surveillance Active**

La base de données nouvellement créée par aideinit n'est pas immédiatement active. Elle est stockée dans aide.db.new.gz et doit être renommée pour devenir la base de référence officielle, aide.db.gz, qu'AIDE utilisera pour ses vérifications.

* **Copiez la nouvelle base de données** :  
  `sudo cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz`

L'utilisation de cp (copier) plutôt que mv (déplacer) lors de la toute première initialisation est une bonne pratique. Elle permet de conserver le fichier .new.gz comme une sauvegarde temporaire, au cas où une vérification immédiate serait nécessaire pour confirmer que le processus s'est bien déroulé. Pour les mises à jour ultérieures, mv est souvent utilisé.

### **4.3. Exécution des Contrôles d'Intégrité Manuels via aide \--check**

Une fois la base de référence en place, vous pouvez lancer une vérification manuelle à tout moment pour comparer l'état actuel du système à l'instantané enregistré.

* **Lancez la vérification** :  
  `sudo aide --check`  
  L'option \-C est un alias pour \--check.

* **Interprétez le résultat** :  
  * **Sur un système non modifié**, la sortie sera concise et rassurante :  
    `AIDE found NO differences between database and filesystem. Looks okay!!`

  * **Pour démontrer le fonctionnement**, effectuons un test simple. Créons un fichier dans un répertoire surveillé, comme /etc :  
    `sudo touch /etc/test_aide_file`  
    Relançons ensuite la vérification :  
    `sudo aide --check`  
    La sortie sera maintenant très différente, indiquant qu'un changement a été détecté et listant le nouveau fichier dans la section "Added entries".

### 

### **4.4. Le Cycle de Vie des Mises à Jour : Gérer les Changements Légitimes avec aide \--update**

Un système d'exploitation n'est jamais complètement statique. Les mises à jour de paquets, les modifications de configuration par l'administrateur et d'autres tâches de maintenance sont des changements légitimes. Si la base de données AIDE n'est pas mise à jour après ces changements, chaque vérification future générera des alertes, créant une fatigue qui mène à ignorer les rapports. Le processus de mise à jour de la base de référence est donc une opération de maintenance critique.

Ce processus est volontairement conçu en deux étapes pour des raisons de sécurité. Il force l'administrateur à examiner les changements détectés avant de les accepter comme le nouvel état "normal" du système. Cela empêche l'acceptation aveugle de modifications qui pourraient inclure une activité malveillante.

1. **Générer une base de données mise à jour** : Après avoir effectué un changement légitime (par exemple, sudo apt upgrade), exécutez la commande aide \--update.  
   `sudo aide --update`  
   ou  
   `sudo aide -u`

Cette commande effectue une vérification complète, similaire à \--check, mais au lieu de simplement afficher un rapport, elle utilise les résultats pour créer une nouvelle base de données dans /var/lib/aide/aide.db.new.gz qui reflète l'état actuel du système. La sortie de cette commande est elle-même un rapport des différences trouvées, que l'administrateur doit examiner attentivement.

2. **Accepter la nouvelle base de référence** : Après avoir examiné la sortie de aide \--update et confirmé que tous les changements listés sont légitimes, l'étape finale consiste à promouvoir la nouvelle base de données en tant que référence active.  
   `sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz`

Cette commande remplace l'ancienne base de référence par la nouvelle. La prochaine exécution de aide \--check utilisera cette nouvelle base et ne signalera plus les changements qui viennent d'être acceptés. Ce cycle "vérifier \-\> modifier \-\> mettre à jour \-\> accepter" est fondamental pour la gestion à long terme d'AIDE.

## 

## **5\. Interprétation des Rapports AIDE pour une Analyse Actionnable**

Un rapport AIDE peut être dense, mais savoir le lire rapidement et efficacement est une compétence cruciale pour tout administrateur système. C'est la capacité à distinguer un changement de routine d'une anomalie potentiellement malveillante qui transforme AIDE d'un simple outil de journalisation en un véritable système de détection d'intrusion.

### **5.1. Déconstruction d'un Rapport AIDE : Sections de Résumé et de Détails**

Un rapport généré par aide \--check est généralement structuré en plusieurs sections claires.

* **En-tête du rapport** : Il contient des informations contextuelles comme l'heure de début de la vérification, les noms des bases de données comparées et le niveau de verbosité.  
* **Section de résumé (Summary)** : C'est la partie la plus importante pour une évaluation rapide. Elle fournit un décompte des changements détectés :

  * Total number of entries : Le nombre total d'objets (fichiers, répertoires) dans la base de données.  
  * Added entries : Le nombre de nouveaux fichiers ou répertoires qui n'existaient pas dans la base de référence.  
  * Removed entries : Le nombre de fichiers ou répertoires qui étaient dans la base de référence mais qui n'existent plus sur le système.  
  * Changed entries : Le nombre de fichiers ou répertoires dont un ou plusieurs attributs surveillés ont changé.


* **Sections détaillées** : Après le résumé, le rapport liste de manière exhaustive chaque fichier concerné, regroupé par catégorie (Ajouté, Supprimé, Modifié). Pour les fichiers modifiés, le rapport détaille précisément quels attributs ont changé, en montrant l'ancienne et la nouvelle valeur côte à côte.Exemple de sortie pour un fichier modifié :  
```
  `Changed entries:`  
  `---------------------------------------------------`  
  `f =.... mc..C.. : /etc/aide/aide.conf`  
  `---------------------------------------------------`  
  `Detailed information about changes:`  
  `---------------------------------------------------`  
  `File: /etc/aide/aide.conf`  
    `Size   : 6598 | 58434`  
    `Mtime  : 2023-05-09 05:46:16 +0000 | 2024-09-10 10:20:30 +0000`  
    `Ctime  : 2023-05-09 05:46:16 +0000 | 2024-09-10 10:20:30 +0000`  
    `SHA512 : old_hash_value... | new_hash_value...`
```

### 

### **5.2. Guide de Décodage des Descripteurs de Changement d'AIDE**

La ligne de résumé pour chaque fichier modifié (ex: f \>.p..-..c...sha512) est une chaîne de caractères codée qui fournit une vue d'ensemble extrêmement dense des changements. Comprendre ce code permet une analyse très rapide.

Cette chaîne de caractères a un format fixe. Chaque position correspond à un attribut spécifique. Un . signifie "pas de changement", un \+ signifie "attribut ajouté", un \- signifie "attribut supprimé", et une lettre indique un changement de cet attribut.

**Table 2: Clé d'Interprétation des Rapports de Changement AIDE**

```
| Position / Symbole | Code | Signification |
| :---- | :---- | :---- |
| 1: Type de Fichier | f, d, l | Fichier (f), Répertoire (d), Lien symbolique (l). Un \! indique un changement de type. |
| 2: Lien Symbolique | l | Le nom du lien a changé. |
| 3: Taille | \>, \<, \= | La taille a augmenté (\>), diminué (\<), ou est restée identique (=). |
| 4: Blocs | b | Le nombre de blocs alloués a changé. |
| 5: Permissions | p | Les permissions (ex: rwx) ont changé. **Alerte potentielle.** |
| 6: Utilisateur (UID) | u | Le propriétaire a changé. **Alerte potentielle.** |
| 7: Groupe (GID) | g | Le groupe propriétaire a changé. **Alerte potentielle.** |
| 8: atime | a | L'heure du dernier accès a changé. (Souvent ignoré) |
| 9: mtime | m | L'heure de la dernière modification du contenu a changé. |
| 10: ctime | c | L'heure du dernier changement de statut a changé. |
| 11: Inode | i | Le numéro d'inode a changé (souvent signe d'un remplacement de fichier). |
| 12: Nb de Liens | n | Le nombre de liens durs a changé. |
| 13: Hachage | H | Au moins une des empreintes cryptographiques a changé. **Indique une modification du contenu.** |
| 14: ACL | A | La liste de contrôle d'accès (ACL) a changé. |
| 15: XAttrs | X | Les attributs étendus ont changé. |
| 16: SELinux | S | Le contexte SELinux a changé. |
```

Avec cette clé, la ligne f \>..p..g..c..H.. peut être rapidement interprétée comme : un fichier (f) dont la taille a augmenté (\>), les permissions ont changé (p), le groupe a changé (g), le ctime a changé (c) et le contenu a changé (H). Une telle ligne pour un fichier dans /bin serait une alerte de sécurité de la plus haute priorité.

### 

### **5.3. Mises en Situation Pratiques : de la Maintenance de Routine à la Détection d'Intrusion**

L'interprétation des rapports prend tout son sens lorsqu'elle est appliquée à des scénarios concrets.

* **Scénario 1 : Analyse post-apt upgrade**  
  * **Rapport attendu** : Un grand nombre de "Changed entries". Les fichiers dans /usr/bin, /usr/lib, etc., afficheront des changements de taille, d'horodatage et de hachage. Des fichiers de configuration dans /etc peuvent également être modifiés.  
  * **Analyse** : Ces changements sont attendus et légitimes. L'administrateur doit vérifier que les fichiers modifiés correspondent bien aux paquets qui ont été mis à jour.  
  * **Action** : Exécuter sudo aide \--update pour créer une nouvelle base de référence, l'examiner, puis l'activer avec sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/[aide.db.gz](http://aide.db.gz).

* **Scénario 2 : Détection d'une modification de /etc/passwd ou /etc/sudoers**  
  * **Rapport attendu** : Une seule "Changed entry". Le descripteur pourrait être f \=....m.c..H.., indiquant une modification du contenu.  
  * **Analyse** : Toute modification non planifiée de ces fichiers est une alerte de sécurité critique. Cela pourrait indiquer une tentative d'ajout d'un utilisateur non autorisé ou d'escalade de privilèges.  
  * **Action** : Isoler immédiatement le système du réseau. Ne PAS mettre à jour la base AIDE. Lancer une investigation forensique complète pour comprendre la nature de la modification et l'origine de la compromission.


* **Scénario 3 : Identification d'un cron malveillant**  
  * **Rapport attendu** : Une "Added entry" pour un nouveau fichier dans /etc/cron.daily/, /etc/cron.d/ ou un autre répertoire de tâches planifiées.  
  * **Analyse** : Un attaquant pourrait ajouter une tâche cron pour maintenir sa persistance sur le système, exfiltrer des données ou lancer une attaque ultérieure. Tout nouveau script dans ces répertoires qui n'a pas été ajouté par l'administrateur ou un paquet légitime est extrêmement suspect.  
  * **Action** : Examiner le contenu du nouveau fichier pour comprendre son objectif. Si malveillant, le supprimer et lancer une investigation pour trouver comment l'attaquant a obtenu les privilèges nécessaires pour l'écrire.


Ces scénarios illustrent comment AIDE, grâce à une interprétation rigoureuse de ses rapports, permet de passer d'une simple observation de changement à une décision de sécurité éclairée.

## 

## **6\. Automatisation, Rapports et Sécurisation Avancée**

Pour que AIDE soit un outil de sécurité efficace, il doit être intégré dans les processus opérationnels du système. Cela implique d'automatiser les vérifications, de configurer des rapports clairs et, surtout, de prendre des mesures pour protéger AIDE lui-même contre toute manipulation.

### **6.1. Automatisation des Vérifications Périodiques avec cron**

L'exécution manuelle des vérifications AIDE est utile pour des contrôles ponctuels, mais la véritable valeur de la surveillance de l'intégrité réside dans sa régularité. L'automatisation via cron est la méthode standard pour y parvenir.

Comme mentionné précédemment, le paquet AIDE sur Ubuntu installe par défaut un script dans /etc/cron.daily/aide. Ce script exécute aide \--check une fois par jour. Pour la plupart des serveurs, cette fréquence est un bon point de départ.

Cependant, pour des besoins plus spécifiques, il est possible de créer une tâche cron personnalisée. Par exemple, pour exécuter une vérification toutes les 12 heures, on peut désactiver ou supprimer le script quotidien et créer une nouvelle entrée dans la crontab de l'utilisateur root.

1. Ouvrez la crontab de root pour l'édition :  
   `sudo crontab -e`

2. Ajoutez la ligne suivante pour une exécution à 4h05 du matin chaque jour :  
   `5 4 * * * /usr/bin/aide --check`

Cette approche offre une flexibilité totale sur la fréquence des analyses, permettant de l'adapter au niveau de criticité du serveur et au volume de changements attendus.

### **6.2. Configuration des Rapports Automatisés par Courriel**

Recevoir les rapports AIDE directement dans sa boîte de réception est le moyen le plus efficace d'être informé rapidement des changements. Plusieurs méthodes permettent de configurer l'envoi de courriels.

* **Méthode 1 : Utilisation de la variable MAILTO de cron (la plus simple)** Le service cron possède une fonctionnalité intégrée qui envoie par courriel toute sortie générée par une tâche planifiée à l'adresse spécifiée dans la variable MAILTO. C'est la méthode la plus directe.Dans la crontab (via sudo crontab \-e), ajoutez la ligne MAILTO avant votre tâche AIDE :  
  `MAILTO="votre.adresse@example.com"`  
  `5 4 * * * /usr/bin/aide --check`

Si des changements sont détectés, le rapport complet généré par aide \--check sera envoyé à l'adresse spécifiée. Si aucun changement n'est détecté, aucun courriel ne sera envoyé (car la commande ne produit pas de sortie, à l'exception d'un message de confirmation qui peut être supprimé avec des options de verbosité).

* **Méthode 2 : Utilisation des fichiers de configuration d'AIDE** Certaines configurations plus anciennes d'AIDE sur Debian/Ubuntu utilisaient un fichier de configuration dans /etc/default/aide pour définir des variables comme MAILTO et MAILSUBJ. Ce fichier est ensuite utilisé par le script /etc/cron.daily/aide. Si vous utilisez le script cron par défaut, vous pouvez éditer ce fichier pour configurer la destination des courriels.

* **Méthode 3 : Redirection manuelle (piping)** Il est également possible de rediriger manuellement la sortie de la commande AIDE vers une commande d'envoi de courriel, comme mail.  
  `5 4 * * * /usr/bin/aide --check | mail -s "Rapport AIDE pour le serveur X" votre.adresse@example.com`

Quelle que soit la méthode choisie, une condition préalable est indispensable : le serveur doit être capable d'envoyer des courriels. Cela nécessite un Agent de Transfert de Courrier (MTA) fonctionnel, comme Postfix, configuré soit pour envoyer les courriels directement, soit pour les relayer via un serveur SMTP externe.

### **6.3. Mesures de Sécurité Essentielles : Protéger la Base de Données et la Configuration**

C'est l'aspect le plus critique et le plus souvent négligé du déploiement d'AIDE. Un système de surveillance de l'intégrité n'est fiable que si ses propres composants sont à l'abri de toute manipulation. Si un attaquant obtient un accès root, il peut non seulement modifier les fichiers système, mais aussi la base de données AIDE pour y intégrer ses modifications, les rendant ainsi "invisibles" aux futures vérifications.  
Plusieurs stratégies, de complexité et de niveau de sécurité croissants, peuvent être mises en œuvre pour contrer cette menace.

* **Stratégie 1 (Sécurité Minimale) : Permissions Restrictives** La mesure la plus simple consiste à s'assurer que les permissions sur la base de données et le fichier de configuration sont aussi restrictives que possible.  
  `sudo chmod 600 /var/lib/aide/aide.db.gz`  
  `sudo chmod 600 /etc/aide.conf`

Cela garantit que seul l'utilisateur root peut lire ces fichiers. C'est une protection contre les utilisateurs non privilégiés, mais inefficace contre un attaquant ayant obtenu les privilèges root.

* **Stratégie 2 (Bonne Sécurité) : Stockage sur un Serveur Centralisé** Une approche plus robuste consiste à copier la base de données de référence (aide.db.gz) sur un serveur distant sécurisé (par exemple, un serveur de logs ou de gestion de configuration). Le script cron quotidien est alors modifié : avant d'exécuter aide \--check, il télécharge la base de référence depuis le serveur distant, l'utilise pour la comparaison, puis envoie le rapport. De cette manière, même si un attaquant modifie la base de données locale, la prochaine vérification utilisera la copie de confiance et détectera la manipulation.

* **Stratégie 3 (Haute Sécurité) : Utilisation de Médias en Lecture Seule** La méthode la plus sécurisée consiste à stocker la base de données de référence, le fichier de configuration aide.conf et même le binaire aide lui-même sur un support physique en lecture seule (comme un CD-ROM, un DVD ou une clé USB avec un interrupteur de protection en écriture). Le processus de vérification devient alors :

  1. Monter le média en lecture seule sur le serveur.  
  2. Exécuter la vérification en utilisant la configuration et la base de données du média.  
  3. Démonter le média. Cette méthode offre la plus haute garantie d'intégrité pour les composants d'AIDE, car il est physiquement impossible pour un attaquant de les modifier. Bien que plus contraignante sur le plan opérationnel, elle est recommandée pour les systèmes les plus critiques.

## **7\. Conclusion et Recommandations Opérationnelles**

Le déploiement de l'Advanced Intrusion Detection Environment (AIDE) sur un serveur Ubuntu 24.04 LTS représente une amélioration significative de la posture de sécurité. En fournissant un mécanisme fiable de surveillance de l'intégrité des fichiers, AIDE offre une visibilité cruciale sur les changements qui affectent les composants critiques du système. Cependant, pour transformer cet outil puissant en un véritable atout de sécurité, il est impératif de l'intégrer dans une stratégie plus large et de suivre des pratiques de maintenance rigoureuses.

### **7.1. Intégration d'AIDE dans une Stratégie de Défense en Profondeur**

Il est fondamental de reconnaître qu'AIDE est un contrôle de détection, et non de prévention. Son rôle est d'alerter sur des modifications *après* qu'elles se sont produites. Par conséquent, son efficacité est décuplée lorsqu'il est intégré dans une architecture de sécurité multicouche. AIDE doit être complété par :

* **Contrôles préventifs** : Des pare-feux (UFW, iptables), des systèmes de contrôle d'accès obligatoires (AppArmor, SELinux), et une politique de renforcement du système (system hardening) pour minimiser la surface d'attaque et empêcher les intrusions initiales.  
* **Autres contrôles de détection** : Une journalisation centralisée (syslog-ng, rsyslog vers un serveur SIEM) pour corréler les alertes d'intégrité de fichiers avec d'autres événements système et réseau, et potentiellement un système de détection d'intrusion réseau (NIDS) pour identifier les activités suspectes en amont.

Ensemble, ces couches créent un système résilient où AIDE agit comme le filet de sécurité final, capable de détecter les activités malveillantes qui auraient pu contourner les défenses préventives.

### 

### **7.2. Synthèse des Bonnes Pratiques pour la Maintenance à Long Terme d'AIDE**

Pour garantir la pertinence et la fiabilité d'AIDE sur le long terme, les recommandations opérationnelles suivantes sont essentielles :

* **Affiner, ne pas ignorer** : Le plus grand danger pour un système FIM est la "fatigue des alertes". Il est impératif de consacrer du temps à l'affinage du fichier aide.conf pour éliminer le bruit généré par les changements légitimes et attendus. Un rapport AIDE qui est systématiquement ignoré en raison de son volume de faux positifs est aussi inutile qu'une absence totale de surveillance.  
* **Intégrer à la Gestion des Changements** : Le processus de mise à jour de la base de données AIDE (aide \--update suivi de l'activation de la nouvelle base) doit devenir une étape standard et obligatoire de toute procédure de gestion des changements. Qu'il s'agisse d'une mise à jour de paquet, d'une modification de configuration ou du déploiement d'une nouvelle application, la mise à jour de la base de référence AIDE doit être la dernière étape qui valide et scelle le nouvel état "sain" du système.  
* **Régénération Périodique de la Base de Référence** : Au-delà des mises à jour incrémentielles, il est recommandé de procéder à une réinitialisation complète de la base de données AIDE à intervalles réguliers (par exemple, tous les trimestres ou après une mise à jour majeure du système d'exploitation). Cette pratique permet de s'assurer que la base de référence ne dérive pas progressivement de l'état réellement attendu du système et de réaffirmer sa fiabilité.  
* **Sécuriser l'Écosystème AIDE** : La confiance que l'on peut accorder aux rapports d'AIDE est directement proportionnelle à la sécurité de ses propres composants. La protection de la base de données de référence, du fichier de configuration et du binaire AIDE contre toute manipulation est une condition non négociable. La mise en œuvre de stratégies de stockage sécurisé (serveur distant ou média en lecture seule) est indispensable pour tout environnement de production critique.

En suivant ces principes, AIDE transcende son rôle d'outil technique pour devenir une composante stratégique de la sécurité, offrant une assurance continue que le système reste dans l'état dans lequel il a été conçu pour être : sécurisé, stable et fiable.

#### 

#### **Sources des citations**

1\. What is an Advanced Intrusion Detection Environment (AIDE) \- Sycurio, https://sycurio.com/knowledge/glossaries/advanced-intrusion-detection-environment-aide 2\. Advanced Intrusion Detection Environment | by Rishabh Umrao \- Medium, https://ayedaemon.medium.com/advanced-intrusion-detection-environment-c0555693a371 3\. AIDE \- ArchWiki, https://wiki.archlinux.org/title/AIDE 4\. en.wikipedia.org, https://en.wikipedia.org/wiki/Advanced\_Intrusion\_Detection\_Environment 5\. AIDE \- Advanced Intrusion Detection Environment, https://aide.github.io/ 6\. Log Analysis: AIDE HIDS \- Pentester Academy Blog, https://blog.pentesteracademy.com/log-analysis-aide-hids-3deafb5e88a6 7\. Install and Configure AIDE on Ubuntu 20.04 \- kifarunix.com, https://kifarunix.com/install-and-configure-aide-on-ubuntu-20-04/ 8\. Chapter 7\. Checking integrity with AIDE | Security hardening | Red Hat Enterprise Linux | 8, https://docs.redhat.com/en/documentation/red\_hat\_enterprise\_linux/8/html/security\_hardening/checking-integrity-with-aide\_security-hardening 9\. Checking Integrity With AIDE \- Fedora Docs, https://docs.fedoraproject.org/en-US/quick-docs/aide-checking-file-integrity/ 10\. How to install AIDE on Ubuntu 22.04 \- Real Time Cloud Services LLC, https://customer.acecloudhosting.com/index.php/knowledgebase/191/How-to-install-AIDE-on-Ubuntu-22.04.html 11\. How to Install AIDE on Ubuntu 20.04 LTS \- VegaStack, https://vegastack.com/tutorials/how-to-install-aide-on-ubuntu-20-04-lts/ 12\. How to Install and Configure AIDE on Ubuntu Linux | Rapid7 Blog, https://www.rapid7.com/blog/post/2017/06/30/how-to-install-and-configure-aide-on-ubuntu-linux/ 13\. AIDE : IDS for Linux Ubuntu Installation and Configuration \- Simplificando Redes, https://simplificandoredes.com/en/aide-ubuntu-installation-configuration/ 14\. How install AIDE on Ubuntu, https://askubuntu.com/questions/1507027/how-install-aide-on-ubuntu 15\. Using Advanced Intrusion Detection Environment \- Oracle Help Center, https://docs.oracle.com/en/operating-systems/oracle-linux/9/security/security-UsingAdvancedIntrusionDetectionEnvironment.html 16\. Using AIDE for file integrity monitoring (FIM) on Ubuntu or Debian \- Stephen R Lang, https://www.stephenrlang.com/2016/03/using-aide-for-file-integrity-monitoring-fim-on-ubuntu/ 17\. update-aide.conf command not found \- Server Fault, https://serverfault.com/questions/1111551/update-aide-conf-command-not-found 18\. aide.conf(5) — aide — Debian jessie, https://manpages.debian.org/jessie/aide/aide.conf.5.en.html 19\. aide.conf \- The configuration file for Advanced Intrusion Detection Environment \- Ubuntu Manpage, https://manpages.ubuntu.com/manpages/jammy/man5/aide.conf.5.html 20\. CIS 1.3.1 Ensure AIDE is installed · Issue \#11929 · ComplianceAsCode/content \- GitHub, https://github.com/ComplianceAsCode/content/issues/11929 21\. aide.conf: The configuration file for Advanced Intrusion Detection Environment | aide File Formats | Man Pages | ManKier, https://www.mankier.com/5/aide.conf 22\. AIDE Manual Version 0.16.2, https://aide.github.io/doc/ 23\. Enhancing Linux security with Advanced Intrusion Detection Environment (AIDE) \- Red Hat, https://www.redhat.com/en/blog/linux-security-aide 24\. aideinit \- create a new AIDE database \- Ubuntu Manpage, https://manpages.ubuntu.com/manpages/noble/man8/aideinit.8.html 25\. UBTU-24-100110 \- Ubuntu 24.04 LTS must configure AIDE to prefo...\<\!-- \--\> | Tenable®, https://www.tenable.com/audits/items/DISA\_STIG\_Canonical\_Ubuntu\_24.04\_LTS\_v1r1.audit:80e2e879dc5375dbe388a6df870d261a 26\. How do I interpret aide.log change summary \- linux \- Server Fault, https://serverfault.com/questions/583987/how-do-i-interpret-aide-log-change-summary 27\. How do I set Cron to send emails? \- command line \- Ask Ubuntu, https://askubuntu.com/questions/1020965/how-do-i-set-cron-to-send-emails 28\. How do I get cron to email me only when AIDE detects file modification? \- Stack Overflow, https://stackoverflow.com/questions/61967646/how-do-i-get-cron-to-email-me-only-when-aide-detects-file-modification 29\. Script and Cronjob Help \- Ubuntu Discourse, https://discourse.ubuntu.com/t/script-and-cronjob-help/54323