# **Mettre en place un laboratoire de détection des menaces sur Ubuntu 24.04 LTS avec chkrootkit, rkhunter et LMD**

## **Section 1: Préparation de l'environnement de laboratoire Ubuntu 24.04**

La mise en place d'un environnement de détection des menaces commence bien avant l'installation des outils de sécurité eux-mêmes. L'efficacité de tout scanner de sécurité dépend entièrement de l'intégrité du système sur lequel il est déployé. Cette section fondamentale établit les prérequis critiques pour garantir que le laboratoire est construit sur une base saine, mise à jour et préparée, un principe essentiel pour toute analyse de sécurité crédible.

### **1.1 Préparation du système : Mises à jour initiales et installation des paquets essentiels**

La première étape, et la plus cruciale, sur une nouvelle installation d'Ubuntu 24.04 LTS est de s'assurer que le système est entièrement à jour. Cette procédure minimise la surface d'attaque en corrigeant les vulnérabilités connues dans le système d'exploitation et ses composants avant même que les outils de sécurité ne soient installés. Le processus implique la mise à jour de l'index des paquets du système et la mise à niveau de tous les paquets installés vers leurs dernières versions disponibles.1 L'exécution de cette étape en premier lieu sécurise la chaîne de confiance ; les outils de sécurité s'appuieront sur le gestionnaire de paquets sous-jacent (apt), et il est donc impératif que le gestionnaire de paquets lui-même et ses dépendances soient sécurisés.

Pour mettre à jour et mettre à niveau le système, exécutez la commande suivante :

```Bash
sudo apt update && sudo apt upgrade \-y
```

Une fois le système à jour, l'étape suivante consiste à installer une suite de paquets essentiels qui seront requis soit par les outils de sécurité eux-mêmes, soit pour les tâches d'investigation ultérieures. Cette préparation proactive garantit que toutes les dépendances sont en place.

* build-essential: Un méta-paquet qui installe le compilateur GCC/G++, make, et d'autres utilitaires nécessaires pour compiler des logiciels à partir des sources. Ceci est particulièrement pertinent pour Linux Malware Detect (LMD), qui est installé à partir des sources.
* git: Un système de contrôle de version distribué, nécessaire pour cloner le code source de LMD depuis son dépôt officiel sur GitHub.
* mailutils: Fournit des programmes pour gérer les e-mails, ce qui est essentiel pour configurer les notifications automatiques par e-mail des alertes de sécurité de rkhunter et LMD. 
* wget: Un utilitaire non interactif pour télécharger des fichiers depuis le web, offrant une méthode alternative pour récupérer les archives sources de LMD.

Installez ces paquets essentiels avec la commande suivante :

```Bash
sudo apt install \-y build-essential git mailutils wget
```

### **1.2 Établissement d'une base de référence sécurisée : Le principe du "Known-Good"**

Un concept fondamental en sécurité informatique, souvent négligé dans les guides de base, est le principe d'établir une base de référence "known-good" (reconnue comme saine). Cela signifie que les outils de sécurité doivent être installés et configurés sur un système dont on a la certitude qu'il n'est pas déjà compromis.

Si un outil comme rkhunter est installé sur un système déjà infecté, sa commande de création de base de référence (--propupd) enregistrera les propriétés (comme les hachages de fichiers) des binaires malveillants comme étant légitimes. Dès lors, l'outil considérera ces fichiers compromis comme "sains" et ne signalera aucune alerte lors des analyses futures, rendant le moniteur de sécurité non seulement inutile mais aussi dangereusement trompeur.

La meilleure pratique consiste donc à effectuer l'installation et la configuration de ces outils de sécurité immédiatement après une nouvelle installation du système d'exploitation. Cette action doit avoir lieu avant que le serveur ne soit exposé au trafic de production, avant le déploiement d'applications complexes et avant que des utilisateurs non fiables n'y aient accès. C'est en respectant cette séquence que l'on établit une "racine de confiance" pour toutes les mesures de sécurité ultérieures. Le laboratoire est ainsi préparé, non seulement avec les bons outils, mais aussi avec l'assurance que sa base de référence initiale est véritablement saine.

## **Section 2: chkrootkit : Analyse fondamentale des rootkits**

chkrootkit (Check Rootkit) est un outil de sécurité classique, écrit en script shell, conçu pour vérifier localement les signes d'une infection par un rootkit. Il constitue une première couche de défense précieuse : un scanner rapide, simple et basé sur des signatures, qui fournit un bilan de santé rapide pour les menaces connues. Il est positionné ici comme un outil de premier passage, utile pour une détection rapide, bien qu'il soit connu pour son potentiel de générer des faux positifs sur les systèmes modernes.

### **2.1 Installation et vérification depuis les dépôts Ubuntu**

chkrootkit est disponible directement depuis les dépôts officiels d'Ubuntu 24.04 (Noble Numbat), ce qui rend son installation simple et directe. L'utilisation du gestionnaire de paquets apt garantit que la version installée a été testée et empaquetée spécifiquement pour la distribution.

Pour installer chkrootkit, exécutez la commande suivante :

```Bash
sudo apt install chkrootkit \-y
```

Après l'installation, il est important de vérifier que l'outil a été installé correctement et de noter sa version. Cela peut être utile pour le dépannage et pour rechercher des problèmes connus spécifiques à cette version. La version peut être vérifiée avec l'option \-V.1 Selon les informations de paquetage pour Ubuntu 24.04, la version attendue est 0.58b ou plus récente.

Exécutez la commande de vérification :

```Bash
chkrootkit \-V
```

### **2.2 Exécution et interprétation des résultats de l'analyse**

L'exécution d'une analyse standard avec chkrootkit se fait en invoquant la commande sans arguments, avec des privilèges root. L'outil effectuera une série de tests sur les binaires système, les processus, les interfaces réseau et les fichiers journaux pour rechercher des signatures et des comportements anormaux associés aux rootkits connus.

Pour lancer une analyse complète, utilisez :

```Bash
sudo chkrootkit
```

La sortie typique est une liste de tests effectués, chacun suivi d'un statut. Les statuts les plus courants sont :

* not infected: Le test n'a trouvé aucun signe d'infection.
* not found: Le fichier ou le composant testé n'a pas été trouvé sur le système. C'est souvent normal pour des services plus anciens ou non installés.  
* INFECTED: L'outil a détecté une signature ou un comportement qui correspond à un rootkit connu. **Cela nécessite une investigation immédiate**, bien que cela puisse être un faux positif.

Pour une utilisation dans des scripts ou pour une journalisation plus propre, chkrootkit propose un mode silencieux (-q). Dans ce mode, seuls les messages "INFECTED" ou les avertissements sont affichés, ce qui facilite grandement l'automatisation et l'analyse des journaux.

Pour une analyse en mode silencieux :

```Bash
sudo chkrootkit \-q
```

### **2.3 Investigation et gestion des faux positifs courants sur Ubuntu 24.04**

La plus grande faiblesse de chkrootkit est sa tendance à générer des faux positifs, en particulier sur les distributions Linux modernes. Un administrateur doit savoir comment identifier et gérer ces alertes pour éviter la "fatigue des alertes", où des avertissements constants et bénins sont finalement ignorés, masquant potentiellement une menace réelle.

Voici une analyse de certains des faux positifs les plus courants sur les systèmes Ubuntu :

* Étude de cas 1 : tcpd INFECTED  
  Ce message est un faux positif largement documenté sur les systèmes basés sur Debian/Ubuntu.13 Le test de chkrootkit pour le rootkit t0rn recherche une chaîne spécifique (/usr/sbin/tcpd) dans les processus en cours d'exécution. Sur les systèmes modernes, des services légitimes peuvent contenir cette chaîne dans leur environnement ou leurs arguments, déclenchant ainsi l'alerte de manière incorrecte.  
* Étude de cas 2 : /sbin/init INFECTED (rootkit Suckit)  
  Cette alerte est un autre faux positif classique qui peut être particulièrement alarmant car il concerne le processus init (PID 1), le père de tous les processus du système.18 Cet avertissement est souvent déclenché par un état incohérent du système, par exemple lorsque des mises à jour du noyau ou de bibliothèques système ont été appliquées mais que le système n'a pas encore été redémarré. Les binaires sur le disque ne correspondent plus aux versions chargées en mémoire, ce qui peut confondre les heuristiques de chkrootkit. La première étape de l'investigation pour cette alerte est de s'assurer que le système est entièrement à jour, puis de le redémarrer et de relancer l'analyse. Dans la plupart des cas, l'avertissement disparaîtra.18  
* Étude de cas 3 : Linux.Xor.DDoS  
  Dans certains cas, chkrootkit peut signaler une infection Linux.Xor.DDoS en pointant vers des fichiers temporaires ou des scripts légitimes. Parfois, il peut même signaler ses propres fichiers ou les fichiers temporaires créés par d'autres outils d'administration système (comme l'installeur de Virtualmin) comme étant malveillants, car ses signatures sont trop larges et correspondent à des motifs de code inoffensifs.

### **2.4 Méthodologie d'investigation avancée des avertissements : Établir l'intégrité des fichiers**

Plutôt que d'ignorer aveuglément les avertissements, une approche professionnelle consiste à les utiliser comme point de départ pour une investigation rigoureuse. La valeur de chkrootkit sur un système moderne ne réside pas tant dans sa capacité à trouver des rootkits avec une précision de 100 %, mais dans sa fonction de "déclencheur" qui impose une vérification formelle de l'intégrité des fichiers système. Un administrateur peut utiliser les outils natifs de Debian/Ubuntu pour vérifier de manière cryptographique si un fichier signalé a été modifié.

1. Identifier le propriétaire du paquet avec dpkg:  
   La première étape consiste à déterminer quel paquet a installé le fichier signalé. La commande dpkg \-S recherche dans la base de données du gestionnaire de paquets pour trouver le propriétaire d'un fichier donné.
   ```Bash  
   dpkg \-S /usr/sbin/tcpd
   ```

   Cette commande confirmera que /usr/sbin/tcpd appartient au paquet tcpd.  

2. Vérifier l'intégrité cryptographique avec debsums:  
   L'outil debsums est un utilitaire puissant qui compare les sommes de contrôle MD5 des fichiers installés sur le système avec les sommes de contrôle originales stockées dans la base de données de dpkg. C'est le moyen le plus fiable de prouver qu'un fichier n'a pas été altéré depuis son installation.

   D'abord, installez debsums s'il n'est pas déjà présent :  
   ```Bash  
   sudo apt install debsums \-y
   ```

   Ensuite, exécutez-le avec les options \-a (pour vérifier tous les fichiers, y compris les fichiers de configuration) et \-s (pour ne signaler que les erreurs). Cela fournit un rapport concis sur les fichiers qui ont échoué à la vérification d'intégrité.  
   ```Bash  
   sudo debsums \-as
   ```
   Si cette commande ne renvoie aucune sortie pour le fichier signalé par chkrootkit, on peut conclure avec un haut degré de certitude que le fichier est intact et que l'alerte était un faux positif.  

3. Alternative : Utiliser dpkg \--verify:  
   Une autre méthode intégrée est la commande dpkg \--verify (ou \-V). Elle effectue une vérification similaire mais son format de sortie est moins direct. Elle signale les différences de somme de contrôle avec un '5' dans la sortie.  
   ```Bash  
   sudo dpkg \--verify tcpd
   ```

   Une sortie vide signifie qu'aucune incohérence n'a été trouvée.

Ce processus d'investigation transforme l'alerte de chkrootkit, même si c'est un faux positif, en une opportunité de renforcer la confiance dans l'intégrité du système.

### **2.5 Automatisation et rapports**

Pour une surveillance continue, les analyses de chkrootkit doivent être automatisées. Il existe deux approches principales pour cela.

La première méthode, intégrée au paquet, consiste à modifier le fichier de configuration de chkrootkit pour activer une tâche cron quotidienne gérée par le paquet lui-même.  
Éditez le fichier /etc/chkrootkit/chkrootkit.conf :

```Bash
sudo nano /etc/chkrootkit/chkrootkit.conf
```

Modifiez la ligne suivante pour l'activer 1 :

```
RUN\_DAILY="true"
```

Vous pouvez également définir des options pour l'exécution quotidienne, comme le mode silencieux :
```
RUN\_DAILY\_OPTS="-q"
```

La deuxième méthode, plus flexible et robuste, consiste à créer une tâche cron personnalisée via la table cron de l'utilisateur root. Cela offre un contrôle plus fin sur l'heure d'exécution, la journalisation et les options.

Ouvrez la table cron de l'utilisateur root :

```Bash
sudo crontab \-e
```

Ajoutez la ligne suivante pour planifier une analyse quotidienne à 3h00 du matin, en utilisant le mode silencieux et en redirigeant la sortie vers un fichier journal daté :

```
0 3 \* \* \* /usr/sbin/chkrootkit \-q \> /var/log/chkrootkit/chkrootkit\_$(date \+\\%Y-\\%m-\\%d).log 2\>&1
```

Cette approche est recommandée car elle crée un historique clair des analyses et isole la sortie de chaque jour, facilitant ainsi l'examen et l'archivage.

## **Section 3: rkhunter : Intégrité avancée du système et chasse aux rootkits**

Si chkrootkit est un outil de reconnaissance rapide, rkhunter (Rootkit Hunter) représente une avancée significative en matière de capacités de détection. Il fonctionne non seulement comme un scanner basé sur des signatures, mais aussi, et c'est là sa plus grande force, comme un puissant moniteur d'intégrité du système. Il y parvient en établissant une base de référence des propriétés des fichiers système et en signalant toute déviation par rapport à cet état "connu et sain". Cette section explore en profondeur l'installation, la configuration et l'utilisation de rkhunter pour en faire une pierre angulaire du laboratoire de sécurité.

### **3.1 Installation et l'étape initiale critique : La création de la base de référence**

Comme chkrootkit, rkhunter est facilement disponible dans les dépôts officiels d'Ubuntu 24.04, ce qui simplifie son installation.

Pour installer rkhunter, exécutez la commande suivante :

```Bash
sudo apt install rkhunter \-y
```

Immédiatement après l'installation, avant même de lancer la première analyse, l'étape la plus critique doit être effectuée : la création de la base de données des propriétés des fichiers. Cette base de référence contient les hachages, les permissions, les tailles et d'autres métadonnées des binaires système essentiels. C'est cette base de données qui permettra à rkhunter de détecter les modifications non autorisées. Il est impératif que cette commande soit exécutée sur un système dont on est certain qu'il est propre.

Pour créer la base de référence initiale, exécutez :

```Bash
sudo rkhunter \--propupd
```

Cette commande doit être exécutée après l'installation initiale et, comme nous le verrons, de manière continue pour maintenir la base de référence à jour après des modifications légitimes du système.

### **3.2 Plongée en profondeur dans la configuration (/etc/rkhunter.conf et /etc/default/rkhunter)**

Une configuration minutieuse est la clé pour transformer rkhunter d'un outil bruyant en un moniteur de sécurité précis et fiable. La configuration est répartie sur deux fichiers principaux.

#### **3.2.1 Configuration principale dans /etc/rkhunter.conf**

Ce fichier volumineux contient la majorité des directives de configuration. Voici les paramètres les plus importants à ajuster pour un serveur Ubuntu 24.04. Ouvrez le fichier pour le modifier :

```Bash
sudo nano /etc/rkhunter.conf
```

* **Mises à jour de la base de données** : Pour que rkhunter puisse télécharger les dernières définitions de menaces, ces options doivent être activées.

```  
  Properties  
  UPDATE\_MIRRORS\=1  
  MIRRORS\_MODE\=0  
  WEB\_CMD\=""
```

  Ces paramètres indiquent à rkhunter d'utiliser plusieurs miroirs de mise à jour et de ne pas utiliser de commande de téléchargement externe spécifique, en s'appuyant sur ses méthodes internes.  
  
* **Intégration avec le gestionnaire de paquets** : C'est l'un des paramètres les plus importants pour les systèmes basés sur Debian/Ubuntu. En définissant PKGMGR sur DPKG, rkhunter utilisera la base de données du gestionnaire de paquets pour vérifier les propriétés des fichiers. Cela réduit considérablement les faux positifs, car l'outil peut distinguer une modification légitime (via une mise à jour de paquet) d'une modification suspecte.  
```
  Properties  
  PKGMGR\=DPKG
```

* **Configuration de sécurité SSH** : rkhunter peut vérifier si la configuration de votre serveur SSH respecte les meilleures pratiques. Il est recommandé d'interdire la connexion root directe via SSH.  
```
  Properties  
  ALLOW\_SSH\_ROOT\_USER\=no
```

  Ce paramètre doit correspondre à la directive PermitRootLogin dans votre fichier /etc/ssh/sshd\_config pour éviter les avertissements inutiles.2  
* **Notifications par e-mail** : Pour une surveillance proactive, rkhunter peut envoyer un e-mail lorsqu'il détecte des avertissements.  

```
  Properties  
  MAIL-ON-WARNING\="votre-email@example.com"
```
  Remplacez l'adresse par celle de l'administrateur système. Un serveur de messagerie local comme Postfix ou un relais SMTP doit être configuré pour que cela fonctionne.

#### **3.2.2 Configuration de l'automatisation dans /etc/default/rkhunter**

Ce fichier contrôle le comportement des scripts cron fournis par le paquet rkhunter.

Ouvrez le fichier pour le modifier :

```Bash
sudo nano /etc/default/rkhunter
```

* **Activer l'analyse quotidienne** : Pour activer le script cron quotidien intégré.  
  ```
  Properties  
  CRON\_DAILY\_RUN\="true"
  ```
* **Activer la mise à jour quotidienne de la base de données** : Pour s'assurer que les définitions de menaces sont toujours à jour.  
    ```
  Properties  
  CRON\_DB\_UPDATE\="true"
    ```

* **Mise à jour automatique de la base de référence après les mises à jour apt** : C'est une fonctionnalité essentielle pour la maintenabilité. Lorsque cette option est activée, rkhunter \--propupd est exécuté automatiquement après chaque transaction apt. Cela maintient la base de référence des propriétés des fichiers synchronisée avec les mises à jour légitimes du système, éliminant ainsi la principale source de faux positifs. Cette intégration transparente avec le cycle de vie du gestionnaire de paquets transforme rkhunter en un moniteur d'intégrité dynamique et intelligent.  
  ```
  Properties  
  APT\_AUTOGEN\="yes"
  ```

Une fois la configuration terminée, il est judicieux de la vérifier pour détecter d'éventuelles erreurs de syntaxe avec la commande :

```Bash
sudo rkhunter \-C
```

### **3.3 Exécution des analyses et interprétation des avertissements**

Avec une configuration correcte, l'exécution des analyses devient une tâche simple.

Pour lancer une analyse complète et interactive :

```Bash
sudo rkhunter \--check
```

Pendant l'analyse, rkhunter fera une pause après chaque section de tests, vous demandant d'appuyer sur "Entrée" pour continuer. Pour les analyses automatisées ou non interactives, utilisez l'option \--skip-keypress (ou \--sk).

Pour n'afficher que les avertissements, ce qui est idéal pour les rapports et les scripts, utilisez l'option \--report-warnings-only (ou \--rwo).

Tous les résultats, qu'ils soient interactifs ou non, sont enregistrés de manière exhaustive dans le fichier journal principal, qui doit être le premier endroit à consulter pour une investigation approfondie :

```Bash
sudo less /var/log/rkhunter.log
```


### **3.4 L'art de la mise en liste blanche : Un guide pratique pour Ubuntu 24.04**

Même avec une configuration soignée, rkhunter peut générer des avertissements qui sont des faux positifs spécifiques à votre environnement. La gestion de ces avertissements se fait par la mise en liste blanche (whitelisting), mais cela doit être fait avec prudence. La méthodologie doit toujours être : **Investiguer \-\> Vérifier \-\> Mettre en liste blanche**. Ne mettez jamais en liste blanche un avertissement sans en comprendre la cause.

Les directives de mise en liste blanche se trouvent dans /etc/rkhunter.conf.

* **ALLOWHIDDENDIR et ALLOWHIDDENFILE** : Certains logiciels légitimes créent des répertoires ou des fichiers cachés qui peuvent déclencher des alertes. Par exemple, Java peut créer /etc/.java. La syntaxe correcte est une entrée par ligne, sans guillemets.

  ```
  Properties  
  ALLOWHIDDENDIR\=/etc/.java  
  ALLOWHIDDENDIR\=/dev/.initramfs
  ```

  Des versions plus anciennes de rkhunter avaient des bogues qui empêchaient la mise en liste blanche correcte des liens symboliques, mais ces problèmes sont résolus dans les versions modernes incluses dans Ubuntu 24.04.34  
* **SCRIPTWHITELIST** : Certains scripts shell, en particulier ceux qui sont obfusqués ou compressés, peuvent être signalés comme suspects. Si après vérification, un script est jugé sûr, il peut être ajouté à la liste blanche.  
  ```
  Properties  
  SCRIPTWHITELIST\=/usr/bin/mon\_script\_personnalise
  ```

* **ALLOWIPCPROC** : Des applications comme les navigateurs web (par exemple, Firefox) utilisent légitimement des segments de mémoire partagée (IPC) de grande taille, ce qui peut déclencher un avertissement. Ces processus peuvent être mis en liste blanche.  
  ```
  Properties  
  ALLOWIPCPROC\=/usr/bin/firefox
  ```

La gestion correcte des listes blanches est un processus itératif. Après la première analyse, investiguez chaque avertissement. Si un avertissement est un faux positif confirmé, ajoutez l'entrée appropriée dans rkhunter.conf, puis relancez l'analyse pour confirmer que l'avertissement a disparu. L'objectif est d'atteindre un état où une analyse "propre" ne génère aucun avertissement, de sorte que toute nouvelle alerte soit immédiatement considérée comme un événement de sécurité digne d'intérêt.

## **Section 4: Linux Malware Detect (LMD) : Analyse active des malwares et réponse**

Alors que chkrootkit et rkhunter se concentrent principalement sur les rootkits et l'intégrité du système d'exploitation, Linux Malware Detect (LMD), également connu sous le nom de maldet, introduit une capacité différente et complémentaire : celle d'un véritable scanner de malwares. Conçu à l'origine pour les environnements d'hébergement web partagés, LMD excelle dans la détection de shells web, de trojans et d'autres types de malwares couramment trouvés dans les répertoires accessibles en écriture par les utilisateurs. Sa caractéristique la plus distinctive est sa capacité de réponse active, allant au-delà de la simple détection pour inclure la mise en quarantaine et le nettoyage des menaces.

### **4.1 Installation depuis la source : Un processus vérifié étape par étape**

Contrairement aux deux outils précédents, LMD n'est pas inclus dans les dépôts officiels d'Ubuntu. Il doit donc être installé manuellement à partir de sa source officielle. Cette méthode garantit que la version la plus récente, avec les dernières signatures et fonctionnalités, est utilisée. Il existe deux méthodes principales pour obtenir les sources : cloner le dépôt Git officiel ou télécharger une archive tarball.36 La méthode Git est généralement préférée car elle facilite le suivi des mises à jour du code source lui-même.

Voici le processus d'installation vérifié en utilisant Git :

1. **Cloner le dépôt GitHub** : Naviguez vers un répertoire temporaire (par exemple, /tmp) et clonez le projet.  
   ```Bash  
   cd /tmp  
   git clone https://github.com/rfxn/linux-malware-detect.git
   ```

2. **Exécuter le script d'installation** : Accédez au répertoire nouvellement créé et exécutez le script d'installation avec les privilèges root.  
   ```Bash  
   cd linux-malware-detect/  
   sudo./install.sh
   ```

   Le script d'installation gère la copie des fichiers dans /usr/local/maldetect, la création des liens symboliques nécessaires (comme /usr/local/sbin/maldet), et la mise en place d'une tâche cron quotidienne.  
3. **Vérifier l'installation** : Confirmez que LMD est correctement installé en vérifiant sa version.  
   ```Bash  
   sudo maldet \--version
   ```

### **4.2 Synergie avec ClamAV : Amélioration des performances d'analyse**

LMD peut s'intégrer à l'antivirus open-source ClamAV pour améliorer considérablement les performances d'analyse des fichiers. Lorsque ClamAV est disponible, LMD l'utilise comme moteur d'analyse, ce qui est souvent plus rapide et plus efficace en termes de ressources que son moteur interne pour les fichiers volumineux.

1. **Installer ClamAV** : Installez les paquets ClamAV nécessaires depuis les dépôts Ubuntu.  
   ```Bash  
   sudo apt install clamav clamav-daemon \-y
   ```

2. **Activer l'intégration dans LMD** : Éditez le fichier de configuration de LMD pour activer l'utilisation de ClamAV.  
   ```Bash  
   sudo nano /usr/local/maldetect/conf.maldet
   ```

   Trouvez la directive scan\_clamscan et changez sa valeur à 1 :  
   ```Properties  
   scan\_clamscan\="1"
   ```

### **4.3 Maîtrise de conf.maldet : Configuration pour une défense proactive**

Le fichier /usr/local/maldetect/conf.maldet est le centre de contrôle de LMD. Une configuration soignée est essentielle pour adapter son comportement aux besoins spécifiques de votre environnement.

* **Alertes par e-mail** : Pour être notifié des résultats de l'analyse.  
  ```
  Properties  
  email\_alert\="1"  
  email\_addr\="votre-email@example.com"
  ```

* **Actions de quarantaine** : C'est la fonctionnalité de réponse active de LMD.  
  * quarantine\_hits="1": Lorsque cette option est activée, tout fichier identifié comme malveillant est immédiatement déplacé vers le répertoire de quarantaine (/usr/local/maldetect/quarantine), le rendant inoffensif.  
  * quarantine\_clean="1": Cette option va plus loin. Pour les menaces connues basées sur des injections de chaînes (par exemple, du code malveillant encodé en base64 injecté dans un fichier PHP légitime), LMD tentera de supprimer l'injection malveillante et de restaurer le fichier nettoyé.  
    Il est recommandé de commencer avec quarantine\_hits="1" et quarantine\_clean="0". Cela permet de contenir les menaces tout en donnant à l'administrateur la possibilité d'examiner les fichiers mis en quarantaine avant de tenter un nettoyage, ce qui évite la perte potentielle de données si le nettoyage échoue ou si l'alerte est un faux positif.
    
* **Mises à jour automatiques** : Pour maintenir l'efficacité de LMD, il est crucial de garder ses signatures et le logiciel lui-même à jour.  
  ```
  Properties  
  autoupdate\_signatures\="1"  
  autoupdate\_version\="1"
  ```
* **Analyse quotidienne automatisée** : Active la tâche cron quotidienne intégrée.  
  ```
  Properties  
  cron\_daily\_scan\="1"
  ```

  Cette tâche est intelligente : elle n'analyse pas l'ensemble du système chaque jour, mais se concentre sur les fichiers qui ont été créés ou modifiés au cours des dernières 24 heures, ce qui est très efficace pour détecter les nouvelles infections.

### **4.4 Méthodologies d'analyse et gestion de la quarantaine**

LMD offre plusieurs façons de lancer des analyses, adaptées à différents scénarios.

* **Analyse complète d'un répertoire** : Pour analyser tous les fichiers dans un chemin spécifique. C'est utile pour une analyse initiale ou hebdomadaire d'un répertoire critique comme /var/www.  
  ```Bash  
  sudo maldet \-a /var/www/
  ```

* **Analyse des fichiers récents** : Pour analyser uniquement les fichiers modifiés au cours des X derniers jours. C'est la méthode utilisée par la tâche cron quotidienne.  
  ```Bash  
  sudo maldet \-r /var/www/
  ```

  Cette commande analysera les fichiers dans /var/www/ qui ont été modifiés au cours des 7 derniers jours.  
* **Consultation des rapports** : Chaque analyse génère un rapport avec un ID unique. Pour consulter un rapport, utilisez cet ID.  
  ```Bash  
  sudo maldet \--report \<SCAN\_ID\>
  ```

  L'ID de l'analyse est affiché à la fin de l'exécution de l'analyse.

La gestion de la quarantaine est une tâche opérationnelle clé. LMD fournit un ensemble complet de commandes pour cela :

* **Mettre en quarantaine les résultats d'une analyse passée** : Si une analyse a été effectuée avec quarantine\_hits="0", vous pouvez mettre en quarantaine ses résultats a posteriori.  
  ```Bash  
  sudo maldet \-q \<SCAN\_ID\>
  ```

* **Nettoyer les résultats d'une analyse** : Pour tenter de nettoyer tous les fichiers malveillants d'un rapport d'analyse.  
  ```Bash  
  sudo maldet \-n \<SCAN\_ID\>
  ```

  ou  
  ```Bash  
  sudo maldet \--clean \<SCAN\_ID\>
  ```

* **Restaurer un fichier** : Si un fichier a été mis en quarantaine par erreur (faux positif) ou après un nettoyage manuel, il peut être restauré à son emplacement d'origine.  
  ```Bash  
  sudo maldet \-s \<NOM\_FICHIER\_EN\_QUARANTAINE\>
  ```

  ou  
  ```Bash  
  sudo maldet \--restore \<NOM\_FICHIER\_EN\_QUARANTAINE\>
  ```

* **Purger les données** : Pour supprimer tous les journaux, les fichiers en quarantaine et les données de session.  
  ```Bash  
  sudo maldet \-p
  ```

  ou  
  ```Bash  
  sudo maldet \--purge
  ```

Cette approche de LMD, qui combine détection et réponse, déplace le rôle de l'administrateur d'une simple investigation à une revue et une prise de décision : "Cette mise en quarantaine était-elle correcte? Dois-je restaurer, nettoyer ou supprimer définitivement ce fichier?". Cela exige un niveau de confiance plus élevé dans la précision de l'outil et une compréhension claire des implications opérationnelles.

### **4.5 Comprendre et personnaliser la tâche cron quotidienne de LMD**

Le script d'installation de LMD place automatiquement un script dans /etc/cron.daily/maldet. Ce script est le moteur de l'automatisation de LMD. Une analyse de son contenu révèle qu'il effectue plusieurs tâches critiques chaque jour :

1. Il vérifie les mises à jour des signatures et de la version de LMD.  
2. Il élague les anciennes données temporaires, de session et de quarantaine (généralement celles de plus de 14 jours).  
3. Si le mode de surveillance n'est pas actif, il lance l'analyse quotidienne des fichiers modifiés au cours des dernières 24 heures dans les répertoires home des utilisateurs.

Une note importante pour les systèmes basés sur Debian/Ubuntu est que le démon cron ignore les scripts dans les répertoires /etc/cron.\* qui contiennent un point (.) dans leur nom.45 Le script d'installation de LMD gère cela correctement en nommant le fichier

maldet, sans extension, garantissant ainsi son exécution.

## **Section 5: Surveillance de sécurité intégrée et analyse stratégique**

Après avoir exploré en détail l'installation, la configuration et l'utilisation de chkrootkit, rkhunter et LMD, cette dernière section synthétise ces connaissances en un cadre stratégique. L'objectif n'est pas de considérer ces outils comme des solutions isolées, mais de les orchestrer en un système de surveillance de la sécurité cohésif et multicouche. Cette approche intégrée permet de maximiser la couverture des menaces et de tirer parti des forces complémentaires de chaque outil.

### **5.1 Analyse comparative : chkrootkit vs. rkhunter vs. LMD**

Chaque outil possède une philosophie de conception, des forces et des faiblesses qui le rendent plus ou moins adapté à des tâches spécifiques. Comprendre ces nuances est essentiel pour déployer une stratégie de défense efficace.

* **chkrootkit** est le plus simple et le plus rapide des trois. Sa force réside dans sa faible dépendance et sa rapidité d'exécution, ce qui en fait un bon candidat pour une vérification rapide et initiale. Cependant, sa base de signatures est moins fréquemment mise à jour et il est très sujet aux faux positifs sur les systèmes modernes, ce qui limite son utilité en tant qu'outil de surveillance principal.  
* **rkhunter** est avant tout un moniteur d'intégrité du système. Sa capacité à créer une base de référence des propriétés des fichiers et à détecter toute déviation est sa caractéristique la plus puissante. Il est excellent pour détecter les modifications non autorisées des fichiers du système d'exploitation, une tactique courante des rootkits. Correctement configuré avec l'intégration dpkg et APT\_AUTOGEN, il devient un gardien très fiable de l'intégrité du système d'exploitation de base.  
* **LMD** est un scanner de malwares spécialisé, avec un accent particulier sur les menaces que l'on trouve couramment dans les environnements web (shells PHP, injecteurs, etc.). Sa principale différenciation est sa capacité de réponse active : il ne se contente pas de détecter, il agit en mettant les menaces en quarantaine. Cela en fait un outil indispensable pour la surveillance des répertoires accessibles en écriture par les utilisateurs, tels que /var/www ou /home.

Le tableau suivant résume cette analyse stratégique :

**Tableau 1 : Comparaison stratégique des outils d'analyse sur Ubuntu**

| Caractéristique | chkrootkit | rkhunter (Rootkit Hunter) | Linux Malware Detect (LMD) |
| :---- | :---- | :---- | :---- |
| **Objectif principal** | Rootkits connus au niveau du noyau et binaires trojanisés | Intégrité du système, rootkits, portes dérobées, renforcement de la configuration | Malwares généraux, shells web (PHP/Perl), trojans, fichiers malveillants |
| **Méthode de détection principale** | Correspondance de signatures, anomalies du système de fichiers/processus | Création de base de référence des propriétés de fichiers (hachages, permissions), signatures, vérifications de configuration | Correspondance de signatures (MD5/HEX), heuristiques, intégration du moteur ClamAV |
| **Source d'installation** | Dépôt APT d'Ubuntu | Dépôt APT d'Ubuntu | Depuis la source (GitHub/Tarball du site web) |
| **Complexité de la configuration** | Faible (chkrootkit.conf est minimal) | Moyenne (Fichier rkhunter.conf complet, nécessite un réglage fin) | Élevée (Fichier conf.maldet complet, nécessite des décisions stratégiques) |
| **Taux de faux positifs** | Élevé, en particulier sur les systèmes modernes | Moyen, significativement réductible avec une configuration appropriée (PKGMGR, APT\_AUTOGEN) | Faible à moyen, généralement plus précis sur ses types de menaces cibles |
| **Capacités de réponse** | Aucune (Détection uniquement) | Aucune (Détection et alerte uniquement) | Élevées (Mise en quarantaine, Nettoyage, Suspension d'utilisateur) |
| **Force principale** | Très rapide, simple, peu de dépendances | Excellent pour détecter les modifications non autorisées des fichiers du SE (surveillance de l'intégrité) | Réponse et confinement actifs, forte concentration sur les menaces basées sur le web |
| **Faiblesse principale** | Signatures obsolètes, taux de faux positifs élevé | Peut être bruyant sans une gestion appropriée des listes blanches et de la base de référence | Ne se concentre pas sur les rootkits au niveau du noyau ; nécessite une compilation |
| **Cas d'utilisation principal** | Une vérification rapide de premier passage pour les rootkits classiques | Surveillance continue de l'intégrité du SE par rapport à une base de référence saine connue | Analyse des répertoires accessibles en écriture par les utilisateurs (par exemple, /var/www, /home) et confinement des menaces |

### **5.2 Développer une stratégie d'analyse cohérente**

Plutôt que de choisir un seul "meilleur" outil, une stratégie de défense en profondeur consiste à les utiliser tous de manière complémentaire et planifiée pour éviter les goulots d'étranglement de performance et maximiser la détection.

* **Quotidiennement** : Exécutez les tâches cron automatisées pour les trois outils, mais à des heures décalées pour répartir la charge. Par exemple : rkhunter à 2h00, chkrootkit à 3h00, et LMD (analyse des fichiers récents) à 4h00. Les rapports quotidiens par e-mail doivent être examinés chaque jour.  
* **Hebdomadairement** : Planifiez une analyse complète et approfondie des répertoires critiques avec LMD, comme le répertoire racine web. Cela peut être accompli avec une tâche cron personnalisée.  
  ```Extrait de code  
  30 1 \* \* 0 /usr/local/sbin/maldet \-a /var/www/ \> /dev/null 2\>&1
  ```

  Cette tâche lancera une analyse complète de /var/www/ chaque dimanche à 1h30 du matin.  
* **À la demande** : Utilisez les outils manuellement pour une investigation forensique après un incident de sécurité suspecté ou pour valider l'état d'un système avant sa mise en production.  
* **Agrégation des journaux** : Pour une gestion centralisée, les journaux (/var/log/chkrootkit/, /var/log/rkhunter.log) et les rapports de LMD doivent être transférés vers un système de gestion des journaux centralisé (par exemple, un serveur syslog-ng, Graylog, ou un SIEM). Cela permet la corrélation des événements entre plusieurs systèmes et la conservation à long terme des preuves.

### **5.3 De l'alerte à l'action : Une check-list de réponse aux incidents de haut niveau**

Lorsqu'un outil génère une alerte positive confirmée, une réponse rapide et structurée est essentielle.

1. **Isoler** : Si la menace est jugée grave (par exemple, un rootkit actif), la première étape est d'isoler la machine du réseau pour empêcher toute propagation ou communication avec un serveur de commande et de contrôle.  
2. **Contenir** : Utilisez les capacités de LMD pour mettre en quarantaine les fichiers malveillants identifiés. Cela permet de neutraliser la menace immédiate tout en préservant les preuves pour une analyse ultérieure.  
3. **Investiguer** : Utilisez des outils comme debsums, dpkg, et une inspection manuelle des fichiers pour comprendre la nature et l'étendue de la compromission. Déterminez le vecteur d'attaque initial si possible.  
4. **Éradiquer** : Supprimez les fichiers malveillants, les processus et tout mécanisme de persistance (par exemple, des tâches cron ou des services systemd suspects).  
5. **Récupérer** : La méthode de récupération la plus fiable et la plus sûre après une compromission confirmée est de reconstruire le système à partir de zéro et de restaurer les données à partir d'une sauvegarde saine connue. Tenter de "nettoyer" un système compromis laisse toujours un risque que des portes dérobées ou des modifications subtiles subsistent.

### **5.4 Reconnaître les limites et les prochaines étapes**

Il est crucial de comprendre que chkrootkit, rkhunter et LMD, bien que puissants, ne sont pas une panacée. Ce sont des outils basés sur l'hôte, principalement réactifs et dépendant de signatures ou d'heuristiques.52 Ils ne peuvent pas détecter les menaces "zero-day" et un attaquant sophistiqué peut tenter de les contourner.

Ces outils doivent faire partie d'une stratégie de sécurité plus large qui inclut :

* Un pare-feu correctement configuré (par exemple, UFW).  
* Des contrôles d'accès utilisateur stricts et le principe du moindre privilège.  
* Une surveillance du réseau (NIDS).  
* Des pratiques de renforcement du système d'exploitation.

Pour faire évoluer la posture de sécurité du laboratoire, les étapes suivantes pourraient inclure le déploiement de systèmes de détection d'intrusion basés sur l'hôte (HIDS) plus avancés comme **AIDE** ou **Samhain**. Ces outils créent une base de données cryptographique détaillée de l'état du système et peuvent détecter des modifications beaucoup plus subtiles que rkhunter.21 À un niveau encore plus avancé, les solutions commerciales de détection et de réponse aux points de terminaison (EDR) offrent une surveillance comportementale en temps réel et des capacités d'investigation bien plus profondes.

En conclusion, la construction de ce laboratoire sur Ubuntu 24.04 avec chkrootkit, rkhunter et LMD fournit une base solide et multicouche pour la détection des menaces au niveau de l'hôte. En comprenant les forces, les faiblesses et les synergies de chaque outil, un administrateur peut passer d'une simple exécution de commandes à une orchestration stratégique de la sécurité, transformant un serveur standard en un environnement activement surveillé et mieux défendu.

#### **Sources des citations**

1. How to Install Chkrootkit on Ubuntu 22.04 \- VegaStack, consulté le septembre 24, 2025, [https://vegastack.com/tutorials/how-to-install-chkrootkit-on-ubuntu-22-04/](https://vegastack.com/tutorials/how-to-install-chkrootkit-on-ubuntu-22-04/)  
2. Comment installer et configurer Rootkit Hunter sur Ubuntu/Debian ..., consulté le septembre 24, 2025, [https://www.webhi.com/how-to/fr/comment-installer-et-configurer-rootkit-hunter-sur-ubuntu-debian/](https://www.webhi.com/how-to/fr/comment-installer-et-configurer-rootkit-hunter-sur-ubuntu-debian/)  
3. How to Install and Use Rkhunter on Ubuntu 22.04 & 20.04 \- TecAdmin, consulté le septembre 24, 2025, [https://tecadmin.net/how-to-install-rkhunter-on-ubuntu/](https://tecadmin.net/how-to-install-rkhunter-on-ubuntu/)  
4. Install Linux Malware Detect on CentOS / Fedora / Ubuntu / Debian ..., consulté le septembre 24, 2025, [https://computingforgeeks.com/install-and-use-linux-malware-detect/](https://computingforgeeks.com/install-and-use-linux-malware-detect/)  
5. (PDF) Full Lab to Master Chkrootkit & rkhunter \- ResearchGate, consulté le septembre 24, 2025, [https://www.researchgate.net/publication/392122974\_Full\_Lab\_to\_Master\_Chkrootkit\_rkhunter](https://www.researchgate.net/publication/392122974_Full_Lab_to_Master_Chkrootkit_rkhunter)  
6. chkrootkit config file options \- Ask Ubuntu, consulté le septembre 24, 2025, [https://askubuntu.com/questions/585437/chkrootkit-config-file-options](https://askubuntu.com/questions/585437/chkrootkit-config-file-options)  
7. How to Install Linux Malware Detect in Ubuntu 20.04 \- Liquid Web, consulté le septembre 24, 2025, [https://www.liquidweb.com/blog/linux-malware-detect-ubuntu-20-04/](https://www.liquidweb.com/blog/linux-malware-detect-ubuntu-20-04/)  
8. Chkrootkit results warning : r/linuxadmin \- Reddit, consulté le septembre 24, 2025, [https://www.reddit.com/r/linuxadmin/comments/389mvb/chkrootkit\_results\_warning/](https://www.reddit.com/r/linuxadmin/comments/389mvb/chkrootkit_results_warning/)  
9. How To Install Chkrootkit on Ubuntu 20.04 OR 22.04 LTS \- YouTube, consulté le septembre 24, 2025, [https://www.youtube.com/watch?v=LluG9ubXIYg](https://www.youtube.com/watch?v=LluG9ubXIYg)  
10. Install chkrootkit on Ubuntu 20.04 \- Lindevs, consulté le septembre 24, 2025, [https://lindevs.com/install-chkrootkit-on-ubuntu](https://lindevs.com/install-chkrootkit-on-ubuntu)  
11. chkrootkit : Noble (24.04) : Ubuntu \- Launchpad, consulté le septembre 24, 2025, [https://launchpad.net/ubuntu/noble/+package/chkrootkit](https://launchpad.net/ubuntu/noble/+package/chkrootkit)  
12. Chkrootkit says rootkits detected? \[SOLVED\] \- (old)Puppy Linux Discussion Forum, consulté le septembre 24, 2025, [https://oldforum.puppylinux.com/viewtopic.php?t=66141](https://oldforum.puppylinux.com/viewtopic.php?t=66141)  
13. rkhunter shows a possible rootkit or a false possitive? \- Ask Ubuntu, consulté le septembre 24, 2025, [https://askubuntu.com/questions/1043922/rkhunter-shows-a-possible-rootkit-or-a-false-possitive](https://askubuntu.com/questions/1043922/rkhunter-shows-a-possible-rootkit-or-a-false-possitive)  
14. Questions sur le rootkit avec chkrootkit et rkhunter : r/linuxquestions \- Reddit, consulté le septembre 24, 2025, [https://www.reddit.com/r/linuxquestions/comments/18mge9a/questions\_about\_rootkit\_with\_chkrootkit\_and/?tl=fr](https://www.reddit.com/r/linuxquestions/comments/18mge9a/questions_about_rootkit_with_chkrootkit_and/?tl=fr)  
15. Do you use anti-malware tools like rkhunter and chkrootkit to secure your Linux machine? Are they relevant/useful for modern linux usage? \- Reddit, consulté le septembre 24, 2025, [https://www.reddit.com/r/linux/comments/82u7v4/do\_you\_use\_antimalware\_tools\_like\_rkhunter\_and/](https://www.reddit.com/r/linux/comments/82u7v4/do_you_use_antimalware_tools_like_rkhunter_and/)  
16. Questions about rootkit with chkrootkit and rkhunter : r/linuxquestions \- Reddit, consulté le septembre 24, 2025, [https://www.reddit.com/r/linuxquestions/comments/18mge9a/questions\_about\_rootkit\_with\_chkrootkit\_and/](https://www.reddit.com/r/linuxquestions/comments/18mge9a/questions_about_rootkit_with_chkrootkit_and/)  
17. chkrootkit shows "tcpd" as INFECTED. Is it a false positive? \- Ask Ubuntu, consulté le septembre 24, 2025, [https://askubuntu.com/questions/883495/chkrootkit-shows-tcpd-as-infected-is-it-a-false-positive](https://askubuntu.com/questions/883495/chkrootkit-shows-tcpd-as-infected-is-it-a-false-positive)  
18. Chkrootkit Suckit rootkit INFECTED message \- What now? \- Dedoimedo, consulté le septembre 24, 2025, [https://www.dedoimedo.com/computers/chkrootkit-suckit-message.html](https://www.dedoimedo.com/computers/chkrootkit-suckit-message.html)  
19. Rkhunter finds suspicious new groups and passwords \- Page 2 \- Manjaro Linux Forum, consulté le septembre 24, 2025, [https://forum.manjaro.org/t/rkhunter-finds-suspicious-new-groups-and-passwords/145192?page=2](https://forum.manjaro.org/t/rkhunter-finds-suspicious-new-groups-and-passwords/145192?page=2)  
20. False positive \`chkrootkit\` on \`slib.sh\` file \- Help\! (Home for newbies) \- Virtualmin Community, consulté le septembre 24, 2025, [https://forum.virtualmin.com/t/false-positive-chkrootkit-on-slib-sh-file/115436](https://forum.virtualmin.com/t/false-positive-chkrootkit-on-slib-sh-file/115436)  
21. How to treat supposed chkrootkit false positive \- Ask Ubuntu, consulté le septembre 24, 2025, [https://askubuntu.com/questions/962260/how-to-treat-supposed-chkrootkit-false-positive](https://askubuntu.com/questions/962260/how-to-treat-supposed-chkrootkit-false-positive)  
22. Can dpkg verify files from an installed package? \- Server Fault, consulté le septembre 24, 2025, [https://serverfault.com/questions/322518/can-dpkg-verify-files-from-an-installed-package](https://serverfault.com/questions/322518/can-dpkg-verify-files-from-an-installed-package)  
23. 14.3. Supervision: Prevention, Detection, Deterrence \- The Debian Administrator's Handbook, consulté le septembre 24, 2025, [https://debian-handbook.info/browse/stable/sect.supervision.html](https://debian-handbook.info/browse/stable/sect.supervision.html)  
24. Installing chkrootkit on a Linux Server \- Hivelocity, consulté le septembre 24, 2025, [https://www.hivelocity.net/kb/install-chkrootkit-on-linux-server/](https://www.hivelocity.net/kb/install-chkrootkit-on-linux-server/)  
25. How to Install RKHunter (RootKit Hunter) On Ubuntu 18.04 \- kifarunix.com, consulté le septembre 24, 2025, [https://kifarunix.com/how-to-install-rkhunter-rootkit-hunter-on-ubuntu-18-04/](https://kifarunix.com/how-to-install-rkhunter-rootkit-hunter-on-ubuntu-18-04/)  
26. Detect Linux Security Holes and Rootkits with Rkhunter | Atlantic.Net, consulté le septembre 24, 2025, [https://www.atlantic.net/dedicated-server-hosting/detect-linux-security-holes-and-rootkits-with-rkhunter-on-ubuntu/](https://www.atlantic.net/dedicated-server-hosting/detect-linux-security-holes-and-rootkits-with-rkhunter-on-ubuntu/)  
27. Rkhunter \- Wiki ubuntu-fr, consulté le septembre 24, 2025, [https://doc.ubuntu-fr.org/rkhunter](https://doc.ubuntu-fr.org/rkhunter)  
28. Sécurisez Linux avec RKHunter efficacement | DevSecOps \- Stephane Robert, consulté le septembre 24, 2025, [https://blog.stephane-robert.info/docs/outils/securite/rkhunter/](https://blog.stephane-robert.info/docs/outils/securite/rkhunter/)  
29. How to Install and Use Rkhunter for Security on Ubuntu 22.04 \- VegaStack, consulté le septembre 24, 2025, [https://vegastack.com/tutorials/how-to-install-and-use-rkhunter-for-security-on-ubuntu-22-04/](https://vegastack.com/tutorials/how-to-install-and-use-rkhunter-for-security-on-ubuntu-22-04/)  
30. RKhunter \- Community Help Wiki, consulté le septembre 24, 2025, [https://help.ubuntu.com/community/RKhunter](https://help.ubuntu.com/community/RKhunter)  
31. Sécurisez vos systèmes Linux avec RKhunter \- IT-Connect, consulté le septembre 24, 2025, [https://www.it-connect.fr/securisez-vos-systemes-linux-avec-rkhunter/](https://www.it-connect.fr/securisez-vos-systemes-linux-avec-rkhunter/)  
32. Rootkit Hunter / Wiki / config2 \- SourceForge, consulté le septembre 24, 2025, [https://sourceforge.net/p/rkhunter/wiki/config2/](https://sourceforge.net/p/rkhunter/wiki/config2/)  
33. Ubuntu \- Rootkit Hunter \- Configuration \- Wiki, consulté le septembre 24, 2025, [https://wiki.sharewiz.net/doku.php?id=ubuntu:rootkit\_hunter:configuration](https://wiki.sharewiz.net/doku.php?id=ubuntu:rootkit_hunter:configuration)  
34. Bug \#883324 “False positive: Hidden file (symbolic link to direc...” : Bugs : rkhunter package : Ubuntu \- Launchpad Bugs, consulté le septembre 24, 2025, [https://bugs.launchpad.net/bugs/883324](https://bugs.launchpad.net/bugs/883324)  
35. Ubuntu 12.04 \+ rkhunter 1.3.8 \= false positives\! \- Digital Cardboard, consulté le septembre 24, 2025, [https://digitalcardboard.com/blog/2012/05/24/ubuntu-12-04-rkhunter-1-3-8-false-positives/](https://digitalcardboard.com/blog/2012/05/24/ubuntu-12-04-rkhunter-1-3-8-false-positives/)  
36. Install Linux Malware Detect on Ubuntu 22.04/Ubuntu 20.04 ..., consulté le septembre 24, 2025, [https://kifarunix.com/install-linux-malware-detect-on-ubuntu-22-04-ubuntu-20-04/](https://kifarunix.com/install-linux-malware-detect-on-ubuntu-22-04-ubuntu-20-04/)  
37. How to Install and Configure maldet (Linux Malware Detect – LMD) \- ServerNoobs, consulté le septembre 24, 2025, [https://www.servernoobs.com/how-to-install-and-configure-maldet-linux-malware-detect-lmd/](https://www.servernoobs.com/how-to-install-and-configure-maldet-linux-malware-detect-lmd/)  
38. Install Linux Malware Detect(LMD) and ClaMav Antivirus | by GNINGHAYE GUEMANDEU Malcolmx Hassler | Medium, consulté le septembre 24, 2025, [https://medium.com/@guemandeuhassler96/install-linux-malware-detect-lmd-and-clamav-antivirus-d9d7e993e86a](https://medium.com/@guemandeuhassler96/install-linux-malware-detect-lmd-and-clamav-antivirus-d9d7e993e86a)  
39. How to Install and Use Linux Malware Detect on Linux \- YouTube, consulté le septembre 24, 2025, [https://www.youtube.com/watch?v=uOwtkzwcZk4](https://www.youtube.com/watch?v=uOwtkzwcZk4)  
40. Automating Maldet scan question \- support.cpanel.net., consulté le septembre 24, 2025, [https://support.cpanel.net/hc/en-us/community/posts/19131557608471-Automating-Maldet-scan-question](https://support.cpanel.net/hc/en-us/community/posts/19131557608471-Automating-Maldet-scan-question)  
41. Maldet (LMD) commands and examples. \- Casbay Knowledgebase, consulté le septembre 24, 2025, [https://www.casbay.com/guide/kb/maldet-lmd-commands-and-examples](https://www.casbay.com/guide/kb/maldet-lmd-commands-and-examples)  
42. How to detect and clean malware from your Linux server using Maldet \- HostPresto\!, consulté le septembre 24, 2025, [https://hostpresto.com/tutorials/how-to-detect-and-clean-malware-from-your-linux-server-using-maldet/?PageSpeed=noscript](https://hostpresto.com/tutorials/how-to-detect-and-clean-malware-from-your-linux-server-using-maldet/?PageSpeed=noscript)  
43. Maldet (LMD) commands and examples \- Freespirits Support Forum, consulté le septembre 24, 2025, [https://www.freespirits.gr/forum/server-security/151-maldet-lmd-commands-and-examples](https://www.freespirits.gr/forum/server-security/151-maldet-lmd-commands-and-examples)  
44. Maldet cron.daily script syntax error \- Stack Overflow, consulté le septembre 24, 2025, [https://stackoverflow.com/questions/32827627/maldet-cron-daily-script-syntax-error](https://stackoverflow.com/questions/32827627/maldet-cron-daily-script-syntax-error)  
45. Why your cron.daily script is not running \- Pete Freitag, consulté le septembre 24, 2025, [https://www.petefreitag.com/blog/cron-daily-not-running/](https://www.petefreitag.com/blog/cron-daily-not-running/)  
46. cron.daily jobs not running \- Ask Ubuntu, consulté le septembre 24, 2025, [https://askubuntu.com/questions/337204/cron-daily-jobs-not-running](https://askubuntu.com/questions/337204/cron-daily-jobs-not-running)  
47. Rootkit Detection — chkrootkit & rkhunter \- Medium, consulté le septembre 24, 2025, [https://medium.com/@rkone1552000/rootkit-detection-chkrootkit-rkhunter-f52394116861](https://medium.com/@rkone1552000/rootkit-detection-chkrootkit-rkhunter-f52394116861)  
48. Compare ClamAV, LMD, Rootkit Hunter, and chkrootkit \- Linux ..., consulté le septembre 24, 2025, [https://linuxsecurity.expert/compare/tools/malware-scanners/](https://linuxsecurity.expert/compare/tools/malware-scanners/)  
49. Anti-malware: "Their" tools VS. "Our" tools. What's the difference? : r/linuxquestions \- Reddit, consulté le septembre 24, 2025, [https://www.reddit.com/r/linuxquestions/comments/1f14r22/antimalware\_their\_tools\_vs\_our\_tools\_whats\_the/](https://www.reddit.com/r/linuxquestions/comments/1f14r22/antimalware_their_tools_vs_our_tools_whats_the/)  
50. 3 antimalware solutions for Linux systems \- Red Hat, consulté le septembre 24, 2025, [https://www.redhat.com/en/blog/3-antimalware-solutions](https://www.redhat.com/en/blog/3-antimalware-solutions)  
51. Dealing with Linux Malware, Insights by the Author of rkhunter, consulté le septembre 24, 2025, [https://linux-audit.com/malware/dealing-with-linux-malware-insights-by-the-author-of-rkhunter/](https://linux-audit.com/malware/dealing-with-linux-malware-insights-by-the-author-of-rkhunter/)  
52. chkrootkit : Jammy (22.04) : Ubuntu \- Launchpad, consulté le septembre 24, 2025, [https://launchpad.net/ubuntu/jammy/+package/chkrootkit](https://launchpad.net/ubuntu/jammy/+package/chkrootkit)
