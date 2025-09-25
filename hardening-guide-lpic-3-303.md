# **Guide de Durcissement Avancé pour Ubuntu Server 20.04 (Inspiré de LPIC-3 303\)**

## **Introduction**

### **Objectif et Audience**

Ce guide constitue une ressource technique approfondie destinée aux administrateurs système et ingénieurs en cybersécurité expérimentés. Son objectif est de détailler les procédures de durcissement avancé pour les serveurs fonctionnant sous Ubuntu 20.04 LTS. En s'appuyant sur les domaines de compétences de la certification LPIC-3 303 "Security", ce document fournit des instructions pratiques, des configurations détaillées et les justifications techniques sous-jacentes pour chaque mesure de sécurité implémentée. Il est conçu pour ceux qui cherchent à élever la posture de sécurité de leur infrastructure au-delà des configurations par défaut, en créant un environnement serveur résilient et défendable.

### **Philosophie de Défense en Profondeur**

La stratégie de sécurité adoptée dans ce guide repose sur le principe de la "défense en profondeur" (Defense in Depth). Cette approche postule qu'aucun mécanisme de sécurité n'est infaillible. Par conséquent, la protection d'un système d'information doit être assurée par une série de couches de défense superposées et redondantes. Si une couche est compromise, les suivantes sont là pour contenir ou stopper l'attaquant. Chaque section de ce guide représente une couche de protection distincte : du durcissement du noyau au contrôle d'accès applicatif, en passant par la surveillance de l'intégrité et la détection d'intrusion réseau. Ces couches, bien que distinctes, sont interconnectées et se renforcent mutuellement pour former une posture de sécurité globale robuste.

### **Alignement LPIC-3 303 et Périmètre**

Le contenu de ce guide est inspiré par la rigueur et l'approche holistique de la certification LPIC-3 303, reconnue comme un standard pour les experts en sécurité Linux. Cependant, pour maintenir une focalisation précise sur le durcissement du système d'exploitation lui-même, ce document exclut délibérément trois domaines importants mais distincts : la configuration sécurisée du service SSH, la gestion du protocole TLS et l'infrastructure à clé publique (PKI) avec les certificats X.509. Cette exclusion permet de consacrer l'intégralité de l'analyse aux mécanismes de protection intrinsèques au serveur Ubuntu, de son noyau à ses services de surveillance.

## 

## **Section 1 : Durcissement du Noyau et des Paramètres Système**

Cette section fondamentale se concentre sur la modification du comportement du noyau Linux au moment de l'exécution (runtime) pour réduire la surface d'attaque au plus bas niveau. Les mesures prises ici sont préventives et visent à neutraliser des classes entières de menaces avant même qu'elles n'atteignent les services applicatifs.

### **1.1. Configuration Avancée de sysctl pour la Sécurité Réseau et Noyau**

L'utilitaire sysctl est l'interface principale pour ajuster les paramètres du noyau Linux sans nécessiter de recompilation. Les configurations sont appliquées via le fichier /etc/sysctl.conf et, de manière plus modulaire, via les fichiers présents dans le répertoire /etc/sysctl.d/. Une configuration inadéquate de ces paramètres peut exposer le système à des attaques par déni de service (DoS), au spoofing d'adresse IP et à la collecte d'informations sensibles par des acteurs malveillants.

#### **Protection de la Pile Réseau IPv4/IPv6**

* **Anti-IP Spoofing :** L'usurpation d'adresse IP est une technique fondamentale utilisée dans de nombreuses attaques réseau. L'activation du filtrage par chemin inverse (Reverse Path Filtering) est une contre-mesure efficace. Ce mécanisme vérifie si la source d'un paquet entrant est atteignable via l'interface sur laquelle il a été reçu. Si ce n'est pas le cas, le paquet est probablement usurpé et est rejeté.  
  `# Active la protection contre l'IP Spoofing`  
  `net.ipv4.conf.default.rp_filter = 1`  
  `net.ipv4.conf.all.rp_filter = 1`

Sous Linux (Ubuntu dans ton cas), les paramètres sysctl peuvent être définis à plusieurs endroits. Le principe est :

* Les fichiers dans **/usr/lib/sysctl.d/** sont ceux fournis par la distribution (ne pas modifier).

* Les fichiers dans **/etc/sysctl.d/** sont ceux destinés à l’administrateur système (prioritaires sur ceux de /usr/lib).

* Le fichier global **/etc/sysctl.conf** est encore accepté mais considéré comme « legacy ».

* Les fichiers sont lus dans l’ordre alphabétique croissant. Si plusieurs fichiers définissent la même clé, **le dernier lu gagne**.

### 

### 

#### **Concernant ton cas**

* Tu as déjà un fichier **99-sysctl.conf** dans /etc/sysctl.d/.

* Comme il est chargé **en dernier** (99 → priorité la plus haute), c’est l’endroit le plus simple et cohérent pour placer tes réglages persistants personnalisés.

Donc pour rp\_filter, tu peux ajouter dans **/etc/sysctl.d/99-sysctl.conf** :

net.ipv4.conf.default.rp\_filter \= 1

net.ipv4.conf.all.rp\_filter \= 1

Puis appliquer sans reboot :

sudo sysctl \--system

#### **Vérification**

sysctl net.ipv4.conf.default.rp\_filter

sysctl net.ipv4.conf.all.rp\_filter

Tu devrais voir \= 1

Ces paramètres demandent au noyau de valider que le chemin de retour pour l'adresse source du paquet correspond à l'interface d'entrée, bloquant ainsi les paquets avec des adresses sources falsifiées.**Scénario de Test Pratique (Anti-IP Spoofing)**

* **Configuration :** Deux VMs sur le même réseau. VM "Attaquant" et VM "Victime" (le serveur durci). L'IP de la victime est 192.168.1.100. L'IP de l'attaquant est 192.168.1.101. Une troisième IP non utilisée sur le réseau est 192.168.1.200.

  * **Action (sur l'attaquant) :** Utilisez hping3 pour forger un paquet ICMP (ping) vers la victime, mais en usurpant l'adresse source 192.168.1.200.  
    `sudo hping3 --icmp -c 1 --spoof 192.168.1.200 192.168.1.100`

  * **Résultat Attendu :** La victime, avec rp\_filter=1, ne répondra pas au ping. Le paquet sera silencieusement rejeté car l'adresse source 192.168.1.200 n'est pas routable via l'interface qui a reçu le paquet. Si rp\_filter était à 0, la victime répondrait (à l'adresse usurpée, pas à l'attaquant).


* **Protection contre les attaques SYN Flood :** Une attaque SYN Flood vise à épuiser les ressources d'un serveur en le submergeant de requêtes de connexion TCP semi-ouvertes. L'activation des TCP SYN Cookies permet au noyau de répondre aux requêtes SYN sans allouer de ressources mémoire dans la table de connexions, en encodant les détails de la connexion dans le numéro de séquence de la réponse SYN-ACK. Le serveur ne crée une entrée dans sa table qu'à la réception du ACK final, validant ainsi la légitimité de la requête.

`# Active les TCP SYN Cookies pour mitiger les attaques SYN Flood`  
`net.ipv4.tcp_syncookies = 1`

Cette mesure rend le serveur beaucoup plus résilient aux attaques par épuisement de ressources.

**Scénario de Test Pratique (SYN Flood)**

* **Configuration :** Mêmes VMs que ci-dessus. Assurez-vous qu'un service est à l'écoute sur un port de la victime (par exemple, un serveur web sur le port 80).


  * **Action (sur l'attaquant) :** Lancez une attaque SYN flood avec hping3.  
    `sudo hping3 -S -p 80 --flood --rand-source 192.168.1.100`

  * **Résultat Attendu :** Sur la victime, pendant l'attaque, vérifiez l'état des connexions avec netstat \-nt | grep SYN\_RECV. Sans tcp\_syncookies, la file d'attente SYN se remplirait rapidement, rendant le service inaccessible. Avec tcp\_syncookies=1, la file d'attente ne se remplit pas et le service reste (plus ou moins) accessible aux clients légitimes, car le noyau gère l'afflux de requêtes sans allouer de ressources.


* **Ignorer les Redirections ICMP :** Les messages de redirection ICMP sont utilisés par les routeurs pour informer un hôte d'un chemin plus direct vers une destination. Cependant, ces messages ne sont pas authentifiés et peuvent être falsifiés par un attaquant pour mener une attaque de type "Man-in-the-Middle" (MitM), en redirigeant le trafic de la victime vers une machine contrôlée par l'attaquant.

`# Ignore les redirections ICMP pour prévenir les attaques MitM`  
`net.ipv4.conf.all.accept_redirects = 0`  
`net.ipv6.conf.all.accept_redirects = 0`

`# Désactive également les redirections "sécurisées" (venant de gateways connues)`  
`net.ipv4.conf.all.secure_redirects = 0`

Sur un serveur, les cas d'utilisation légitimes de cette fonctionnalité sont extrêmement rares, et sa désactivation est une bonne pratique de sécurité.

**Scénario de Test Pratique (ICMP Redirect)**

* **Configuration :** Mêmes VMs. L'IP de la passerelle légitime est 192.168.1.1.  
  * **Action (sur l'attaquant) :** Utilisez netwox pour envoyer un faux message de redirection ICMP à la victime, lui disant que la nouvelle passerelle pour atteindre 8.8.8.8 est l'IP de l'attaquant (192.168.1.101).

    
    `sudo netwox 86 --ip4-dst 192.168.1.100 --ip4-src 192.168.1.1 --ip4-gw 192.168.1.101 --ip4-redirect-dst 8.8.8.8`

  * **Résultat Attendu :** Sur la victime, consultez la table de routage avec ip route. Si le durcissement est efficace (accept\_redirects \= 0), aucune nouvelle route via 192.168.1.101 n'apparaîtra. Si le paramètre était activé, une nouvelle entrée de routage empoisonnée serait visible.


* **Journalisation des Paquets Suspects :** Le noyau peut identifier certains paquets comme étant anormaux ou "martiens" (par exemple, des paquets avec une adresse source non routable comme 0.0.0.0 arrivant sur une interface publique). La journalisation de ces paquets peut aider à détecter des tentatives de scan ou de spoofing.

`# Journalise les paquets avec des adresses sources impossibles`  
`net.ipv4.conf.all.log_martians = 1`

Ces logs, visibles via la commande dmesg, fournissent des indicateurs précoces d'activité réseau anormale.

#### **Renforcement de la Mémoire du Noyau**

* **ASLR (Address Space Layout Randomization) :** L'ASLR est une mesure de protection qui rend les attaques de type "buffer overflow" et autres corruptions de mémoire beaucoup plus difficiles à exploiter. En randomisant les adresses de base de la pile, du tas et des bibliothèques partagées, il empêche un attaquant de prédire l'emplacement du code malveillant qu'il souhaite exécuter.

`# Active l'ASLR avec une forte entropie`  
`kernel.randomize_va_space = 2`

La valeur 2 offre le niveau de randomisation le plus élevé, incluant la randomisation de la zone brk.

* **Restriction d'accès au noyau :** Les messages du noyau, accessibles via dmesg, peuvent contenir des informations sensibles sur le matériel, les pilotes chargés et les erreurs système, qui pourraient être utiles à un attaquant pour profiler le système.

`# Empêche les utilisateurs non privilégiés de lire les messages du noyau via dmesg`  
`kernel.dmesg_restrict = 1`

Cette mesure limite la fuite d'informations vers des utilisateurs non autorisés.

* **Désactivation des Core Dumps SUID :** Un "core dump" est une image de la mémoire d'un processus au moment où il a planté. Si un processus s'exécutant avec des privilèges élevés (setuid) génère un core dump, ce fichier pourrait contenir des informations sensibles (clés, mots de passe, etc.) en mémoire.

`# Empêche les processus setuid/setgid de générer des core dumps`  
`fs.suid_dumpable = 0`

Ceci est une précaution essentielle pour éviter la fuite de données confidentielles.  
Le durcissement via sysctl ne doit pas être vu comme une simple liste de paramètres à appliquer. Il s'agit d'une stratégie de filtrage de première ligne. En configurant le noyau pour qu'il rejette nativement les paquets malformés, les tentatives de spoofing ou les redirections ICMP malveillantes, on réduit considérablement le volume de trafic suspect qui atteint les couches supérieures. Un système de détection d'intrusion réseau (NIDS) comme Suricata ou Snort, qui sera abordé plus loin, bénéficie directement de ce filtrage. Moins d'alertes de bas niveau et de "bruit" de fond signifient que les ressources du NIDS sont préservées pour l'analyse de menaces plus complexes et que les analystes de sécurité ne sont pas submergés par des alertes de faible priorité. C'est une application directe du principe de défense en profondeur : la couche du système d'exploitation protège et optimise la couche de surveillance réseau.

### **1.2. Réduction de la Surface d'Attaque du Noyau**

Chaque module du noyau, protocole réseau ou système de fichiers chargé représente une surface d'attaque potentielle. Une vulnérabilité dans un pilote de périphérique rarement utilisé peut suffire à compromettre l'ensemble du système. Le principe du moindre privilège s'applique donc également au noyau : ne charger que ce qui est strictement nécessaire au fonctionnement du serveur. La méthode la plus propre pour empêcher le chargement d'un module est de créer un fichier de configuration dans /etc/modprobe.d/.

#### **Désactivation des Modules Inutiles**

* **Protocoles Réseau Exotiques :** Les serveurs standards n'ont généralement pas besoin de protocoles de transport spécialisés.

`# /etc/modprobe.d/uncommon-net.conf`  
`install dccp /bin/true`  
`install sctp /bin/true`  
`install rds /bin/true`  
`install tipc /bin/true`

Cette configuration empêche le chargement des modules dccp, sctp, rds et tipc.

* **Systèmes de Fichiers Inutilisés :** Désactiver la prise en charge des systèmes de fichiers qui ne seront jamais utilisés sur le serveur réduit la complexité du noyau et les vecteurs d'attaque potentiels.

`# /etc/modprobe.d/uncommon-fs.conf`  
`install cramfs /bin/true`  
`install freevxfs /bin/true`  
`install jffs2 /bin/true`  
`install hfs /bin/true`  
`install hfsplus /bin/true`  
`install udf /bin/true`

Cette liste désactive les pilotes pour des systèmes de fichiers souvent liés à des médias optiques ou à d'autres systèmes d'exploitation.

* **Périphériques Matériels Non Pertinents :** Un serveur n'a généralement pas besoin de support pour le Bluetooth, les lecteurs de disquettes ou les haut-parleurs internes.

`# /etc/modprobe.d/uncommon-hw.conf`  
`install bluetooth /bin/true`  
`install floppy /bin/true`  
`install pcspkr /bin/true`  
`install thunderbolt /bin/true`

Forcer sans attendre le prochain boot avec `update-initramfs -u`

#### **Vérification réelle**

`Tu peux confirmer que le module n’est pas présent dans le noyau avec :`

`lsmod | grep rds`

`ou bien :`

`cat /proc/modules | grep rds`

Si aucune sortie → le module n’est **pas chargé** ✅

La désactivation la plus critique est souvent celle du stockage USB. Sur un serveur en production, l'accès physique via USB doit être strictement contrôlé.  
`# /etc/modprobe.d/usb-storage.conf`  
`install usb-storage /bin/true`

Cette mesure empêche le montage automatique de tout périphérique de stockage USB.  
**Scénario de Test Pratique (Désactivation de module)**

* **Configuration :** Un serveur durci avec le module usb-storage désactivé.  
  * **Action :** Branchez physiquement une clé USB sur le serveur.  
  * **Résultat Attendu :** Sur le serveur, exécutez lsusb pour voir si le périphérique est détecté au niveau du bus USB, puis dmesg et lsblk pour vérifier si le module usb-storage a été chargé et si un nouveau périphérique bloc (/dev/sdX) est apparu. Si le durcissement est efficace, le périphérique peut être listé par lsusb, mais il ne sera pas reconnu comme un périphérique de stockage et aucune entrée n'apparaîtra dans lsblk.


Cependant, une mesure aussi restrictive que la désactivation de usb-storage ne peut être prise isolément. Un guide de durcissement avancé doit anticiper ses conséquences opérationnelles. La question immédiate qui se pose est : "Comment un administrateur pourra-t-il intervenir en cas d'urgence avec une clé USB de maintenance?". La simple désactivation du module est une approche incomplète. La bonne pratique consiste à coupler cette mesure avec une stratégie de gestion des accès physiques. Cela peut impliquer la mise en place d'un outil comme **USBGuard**, qui permet de créer des listes blanches pour autoriser uniquement des périphériques USB spécifiques (par exemple, un clavier, une clé de sécurité YubiKey ou une clé de maintenance chiffrée dont l'identifiant est connu). Une autre alternative est de s'assurer de la disponibilité d'un accès console distant sécurisé et hors bande, comme un KVM-over-IP, qui rend l'utilisation de périphériques USB physiques superflue pour la plupart des tâches de maintenance.

### **1.3. Sécurisation des Composants systemd**

systemd est le gestionnaire de système et de services pour la majorité des distributions Linux modernes, y compris Ubuntu. Le durcissement de ses composants est essentiel car il contrôle le démarrage des services, la journalisation, les sessions utilisateur et plus encore.

* **systemd-journald (Journalisation) :** Pour garantir que les journaux d'événements survivent à un redémarrage (ce qui est crucial pour l'analyse post-mortem d'un incident), le stockage persistant doit être activé.  
  `# /etc/systemd/journald.conf`  
  `Storage=persistent`  
  `ForwardToSyslog=yes`  
  `Compress=yes`

L'option ForwardToSyslog=yes facilite l'intégration avec des systèmes de journalisation centralisés (comme un SIEM), et Compress=yes permet d'économiser de l'espace disque.

* **systemd-logind (Gestion des Sessions) :** Pour éviter que des processus lancés par un utilisateur ne continuent de s'exécuter après sa déconnexion (processus "orphelins"), il est recommandé de terminer automatiquement tous les processus de l'utilisateur à la fin de sa session.  
  `# /etc/systemd/logind.conf`  
  `KillUserProcesses=yes`  
  `RemoveIPC=yes`  
  `IdleAction=lock`  
  `IdleActionSec=15min`

RemoveIPC=yes nettoie les objets de communication inter-processus, tandis que les options IdleAction permettent de verrouiller automatiquement les sessions inactives.

* **systemd-resolved (Résolution DNS) :** Pour améliorer la confidentialité et l'intégrité des requêtes DNS, il est possible d'activer DNS-over-TLS (DoT) et DNSSEC.  
  `# /etc/systemd/resolved.conf`  
  `DNSOverTLS=opportunistic`  
  `DNSSEC=allow-downgrade`

opportunistic signifie que resolved tentera d'utiliser DoT si le serveur DNS le supporte. DNSSEC=allow-downgrade tente la validation DNSSEC mais ne fait pas échouer la résolution si elle n'est pas disponible, offrant un bon compromis entre sécurité et compatibilité.

* **Limites de Ressources par Défaut :** Pour prévenir les attaques par épuisement de ressources comme les "fork bombs", il est judicieux de définir des limites globales pour tous les services gérés par systemd.  
  `# /etc/systemd/system.conf`  
  `DefaultLimitCORE=0`  
  `DefaultLimitNOFILE=1024`  
  `DefaultLimitNPROC=1024`  
  `DumpCore=no`

* DefaultLimitCORE=0 et DumpCore=no désactivent la création de core dumps pour tous les services, tandis que DefaultLimitNOFILE (nombre de fichiers ouverts) et DefaultLimitNPROC (nombre de processus) doivent être ajustés en fonction des besoins des applications hébergées sur le serveur.

## 

## **Section 2 : Sécurisation des Systèmes de Fichiers et des Partitions**

Cette section aborde la structure physique et logique du stockage. La manière dont les systèmes de fichiers sont partitionnés et montés a un impact direct sur la capacité d'un attaquant à exécuter du code, à élever ses privilèges ou à provoquer un déni de service.

### **2.1. Stratégies de Partitionnement Sécurisé**

Isoler les répertoires système clés sur des partitions distinctes est une pratique de sécurité fondamentale. Cette séparation offre deux avantages majeurs : le **confinement** et la **spécificité**. Le confinement empêche un problème sur une partition (par exemple, un répertoire /tmp saturé par une application malveillante) de paralyser l'ensemble du système en remplissant la partition racine (/). La spécificité permet d'appliquer des options de montage de sécurité restrictives à chaque partition en fonction de son rôle.

Les partitions suivantes devraient être créées séparément lors de l'installation du système ou ajoutées ultérieurement à l'aide de LVM (Logical Volume Management) :

* **/tmp :** Destiné aux fichiers temporaires. L'isoler est crucial pour pouvoir le monter avec l'option noexec, empêchant l'exécution de tout binaire depuis ce répertoire, un vecteur d'attaque courant.  
* **/var/tmp :** Pour les fichiers temporaires qui doivent persister entre les redémarrages. Une fois /tmp sécurisé, il est courant de créer un lien symbolique de /var/tmp vers /tmp pour appliquer les mêmes protections.  
* **/home :** Contient les répertoires personnels des utilisateurs. Le séparer empêche les utilisateurs de saturer la partition racine avec leurs données et permet d'appliquer les options nodev et nosuid pour bloquer la création de fichiers de périphériques et l'exécution de binaires SUID.  
* **/var/log et /var/log/audit :** Isoler les journaux système et les journaux d'audit protège contre les attaques de type "log flooding", où un attaquant tente de saturer le disque en générant une quantité massive de logs. Cela garantit que la journalisation peut continuer même si d'autres partitions sont pleines.

Une approche moderne pour /tmp consiste à utiliser un système de fichiers en mémoire (tmpfs) géré par systemd, plutôt qu'une partition physique sur le disque. Cela peut être accompli en créant une unité de montage systemd (/etc/systemd/system/tmp.mount) et en s'assurant que /tmp n'est pas listé dans /etc/fstab. Cette méthode offre d'excellentes performances et le contenu de /tmp est automatiquement effacé à chaque redémarrage.

### **2.2. Application des Options de Montage Restrictives**

Le fichier /etc/fstab est plus qu'un simple fichier de configuration ; c'est un puissant mécanisme de contrôle d'accès au niveau du système de fichiers. Les options de montage qui y sont définies dictent la manière dont le noyau interprète les données sur une partition, permettant d'appliquer des politiques de sécurité granulaires.

Les trois options de sécurité les plus importantes sont :

* **nodev :** Empêche l'interprétation des fichiers de type "périphérique" (character ou block devices). Un attaquant ne pourra pas créer un fichier pointant vers /dev/sda sur une partition montée avec nodev pour tenter de lire le disque brut. Cette option est essentielle pour toutes les partitions sauf la racine (/).

* **nosuid :** Demande au noyau d'ignorer les bits setuid et setgid sur les fichiers exécutables. Ces bits permettent à un programme de s'exécuter avec les permissions de son propriétaire (souvent root) plutôt que celles de l'utilisateur qui le lance. Bloquer cette fonctionnalité sur les partitions de données comme /home ou /tmp élimine un vecteur commun d'élévation de privilèges.

* **noexec :** L'option la plus restrictive. Elle empêche l'exécution de tout programme binaire depuis la partition. Elle est idéale pour les partitions qui ne doivent contenir que des données, comme /tmp, /var/log ou /dev/shm. Attention, l'appliquer sur /home peut casser certaines applications légitimes (scripts, applications de développement).

Le tableau suivant synthétise les options de montage recommandées. Les administrateurs peuvent l'utiliser comme une référence fiable pour configurer /etc/fstab, réduisant ainsi le risque d'erreur tout en appliquant une politique de sécurité cohérente et justifiée.

| Point de Montage | Options Recommandées | Justification des Risques Mitigés |
| :---- | :---- | :---- |
| /tmp | defaults,rw,nosuid,nodev,noexec | Empêche l'exécution de malwares téléchargés, l'élévation de privilèges via des binaires SUID et la création de fichiers de périphériques malveillants. |
| /home | defaults,rw,nosuid,nodev | Empêche les utilisateurs d'exécuter des binaires SUID pour élever leurs privilèges et de créer des fichiers de périphériques. noexec est souvent trop restrictif ici. |
| /var | defaults,rw,nosuid | L'option nosuid est une bonne précaution. noexec n'est généralement pas possible car /var peut contenir des scripts pour certains services. |
| /dev/shm | defaults,rw,nosuid,nodev,noexec | Sécurise la mémoire partagée contre l'exécution de code, l'élévation de privilèges et la création de fichiers de périphériques, transformant cet espace en une zone de données uniquement. |

**Scénario de Test Pratique (Option noexec)**

* **Configuration :** Un accès (par exemple, un shell SSH à faibles privilèges) au serveur durci.  
* **Action :** Créez un script shell simple dans /tmp, rendez-le exécutable et essayez de le lancer.  
  `# Sur le serveur durci`  
  `echo '#!/bin/bash' > /tmp/test_exec.sh`  
  `echo 'echo "Execution reussie!"' >> /tmp/test_exec.sh`  
  `chmod +x /tmp/test_exec.sh`  
  `/tmp/test_exec.sh`

* **Résultat Attendu :** La dernière commande échouera avec un message d'erreur bash: /tmp/test\_exec.sh: Permission denied. Cela confirme que l'option noexec sur la partition /tmp empêche bien l'exécution de binaires.

### **2.3. Verrouillage de la Mémoire Partagée (/dev/shm) et du Système de Fichiers /proc**

Deux systèmes de fichiers virtuels méritent une attention particulière en matière de durcissement.

* **Mémoire Partagée (/dev/shm) :** Ce répertoire est une implémentation de la mémoire partagée POSIX, montée en tant que tmpfs. Il permet un échange de données très rapide entre les processus. Cependant, par défaut, il est souvent monté avec des permissions qui permettent à un attaquant d'y stocker et d'y exécuter des malwares. La sécurisation est simple et très efficace. Il suffit de modifier sa ligne dans /etc/fstab :  
  `# Ligne originale (ou similaire) :`  
  `# tmpfs /dev/shm tmpfs defaults 0 0`  
  `# Ligne sécurisée :`  
  `tmpfs /dev/shm tmpfs defaults,nosuid,nodev,noexec 0 0`

Après avoir modifié /etc/fstab, la commande sudo mount \-o remount /dev/shm applique les nouvelles options sans nécessiter de redémarrage.

* **Système de Fichiers /proc :** Ce système de fichiers virtuel expose une grande quantité d'informations sur les processus et le noyau. Par défaut, n'importe quel utilisateur peut lister tous les processus en cours sur le système, ce qui permet à un attaquant de faire de la reconnaissance, d'identifier les services qui tournent, les utilisateurs connectés et les privilèges associés. L'option de montage hidepid=2 est une mesure de confinement extrêmement efficace.

`# Dans /etc/fstab, modifier la ligne pour /proc :`  
`proc /proc proc defaults,hidepid=2 0 0`

Avec hidepid=2, un utilisateur ne peut voir que ses propres processus. Les processus des autres utilisateurs sont invisibles. Cela entrave considérablement la capacité d'un attaquant, même après avoir obtenu un premier accès avec un compte à faibles privilèges, à comprendre l'environnement et à planifier ses prochains mouvements (mouvement latéral).

## 

## **Section 3 : Contrôle d'Accès Obligatoire avec AppArmor**

Le contrôle d'accès obligatoire (Mandatory Access Control, MAC) est un mécanisme de sécurité qui surpasse le modèle de permissions Unix traditionnel (Discretionary Access Control, DAC). Avec le MAC, une politique de sécurité centrale, définie par l'administrateur, dicte ce qu'une application est autorisée à faire, indépendamment des permissions de l'utilisateur qui la lance. AppArmor est l'implémentation MAC par défaut d'Ubuntu, offrant un confinement puissant et granulaire pour les applications.

### **3.1. Principes et Gestion des Profils AppArmor**

AppArmor est un module de sécurité du noyau Linux (LSM) qui fonctionne sur la base des chemins de fichiers (path-based). Il est installé et activé par défaut sur Ubuntu 20.04. Sa politique de sécurité est définie dans des fichiers texte appelés "profils", situés dans /etc/apparmor.d/. Chaque profil est nommé d'après le chemin complet de l'exécutable qu'il confine, en remplaçant les / par des . (par exemple, le profil pour /usr/sbin/ntpd se nomme usr.sbin.ntpd).

#### **Modes d'Opération**

AppArmor fonctionne principalement dans deux modes pour chaque profil :

* **enforce (ou confined) :** C'est le mode de production. Toute action d'une application qui viole les règles de son profil est bloquée par le noyau et l'événement est journalisé. C'est ce mode qui assure le confinement effectif.

* **complain (ou learning) :** Dans ce mode, les violations de profil ne sont pas bloquées, mais elles sont journalisées. Ce mode est indispensable pour développer ou déboguer un profil. Il permet d'observer le comportement d'une application et de construire les règles nécessaires sans interrompre son fonctionnement.

#### **Gestion des Profils avec les Utilitaires aa-\***

Le paquet apparmor-utils fournit une suite d'outils en ligne de commande pour gérer AppArmor :

* **sudo aa-status (ou sudo apparmor\_status) :** Affiche l'état actuel d'AppArmor. Il liste les profils chargés, indique s'ils sont en mode enforce ou complain, et montre les processus actuellement confinés par chaque profil.  
* **sudo aa-enforce /etc/apparmor.d/profil.name :** Passe le profil spécifié en mode enforce. On peut utiliser un glob (\*) pour appliquer le mode à tous les profils.  
* **sudo aa-complain /etc/apparmor.d/profil.name :** Passe le profil spécifié en mode complain.  
* **sudo apparmor\_parser \-r /etc/apparmor.d/profil.name :** Recharge un profil dans le noyau après y avoir apporté des modifications. C'est l'équivalent d'un "refresh" pour une politique AppArmor.

### **3.2. Création d'un Profil Personnalisé : Tutoriel Pratique**

De nombreuses applications, en particulier celles installées manuellement à partir des sources ou des logiciels tiers, ne sont pas fournies avec un profil AppArmor. En créer un est une étape de durcissement proactive et très efficace. Le processus est itératif et semi-automatisé.

* **Étape 1 : Génération du Squelette de Profil avec aa-genprof**  
  1. Lancez l'application pour laquelle vous souhaitez créer un profil (par exemple, un serveur web personnalisé).  
  2. Dans un autre terminal, exécutez la commande sudo aa-genprof /chemin/vers/executable.  
  3. L'outil vous demandera de scanner les journaux système pour les événements AppArmor. Appuyez sur S (Scan).  
  4. Retournez à votre application et utilisez-la de manière exhaustive. Exercez toutes ses fonctionnalités : lecture de fichiers de configuration, écoute sur des ports réseau, écriture dans des fichiers de log, accès à une base de données, etc. Chaque action génère des événements d'accès que le noyau enregistre.


* **Étape 2 : Analyse des Événements et Création des Règles**  
  1. Retournez au terminal où aa-genprof est en cours d'exécution et appuyez de nouveau sur S.  
  2. L'outil va maintenant analyser les journaux d'audit (typiquement dmesg ou /var/log/audit/audit.log) et vous présenter chaque événement de violation détecté.  
  3. Pour chaque événement, aa-genprof vous proposera une ou plusieurs options pour créer une règle. Par exemple, s'il détecte que l'application a tenté de lire /etc/myapp.conf, il pourrait vous demander : (A)llow, (D)eny, (I)gnore, (N)ew, (G)lob,...?. Vous devez choisir l'option la plus appropriée. Allow créera une règle spécifique pour ce fichier, tandis que Glob pourrait vous aider à créer une règle plus générique comme /etc/myapp.\*.


* **Étape 3 : Affinage du Profil avec aa-logprof**  
  1. Une fois que vous avez traité tous les événements dans aa-genprof, sauvegardez et quittez (F pour Finish). Le profil de base est créé dans /etc/apparmor.d/ et est automatiquement mis en mode complain.  
  2. Continuez à utiliser l'application dans des conditions normales.  
  3. Périodiquement, exécutez sudo aa-logprof. Cet outil, similaire à aa-genprof, scanne les journaux à la recherche de nouvelles violations et vous propose de mettre à jour le profil existant. C'est un processus itératif qui permet d'affiner le profil jusqu'à ce qu'aucune nouvelle violation ne soit détectée lors d'une utilisation normale.


* **Étape 4 : Passage en Mode enforce**  
  1. Lorsque vous êtes certain que le profil est complet et que l'application fonctionne correctement sans générer de logs de violation en mode complain, il est temps de l'activer.  
  2. Exécutez sudo aa-enforce /etc/apparmor.d/chemin.vers.executable.  
  3. Vérifiez avec sudo aa-status que le profil est bien passé en mode enforce.


**Scénario de Test Pratique (Profil AppArmor)**

* **Configuration :** Créez un profil AppArmor simple pour ping qui lui interdit d'accéder à un fichier spécifique. Créez le fichier /etc/apparmor.d/bin.ping avec le contenu suivant :  
  `#include <tunables/global>`  
  `/bin/ping flags=(enforce) {`  
    `#include <abstractions/base>`  
    `#include <abstractions/nameservice>`  
    `capability net_raw,`  
    `/bin/ping mr,`  
    `deny /etc/secret.txt r,`  
  `}`

* Rechargez le profil : sudo apparmor\_parser \-r /etc/apparmor.d/bin.ping. Créez le fichier secret : sudo touch /etc/secret.txt.

* **Action :** Utilisez strace avec ping pour tenter de lire le fichier interdit.  
  `strace -e openat ping -c 1 localhost 2>&1 | grep secret.txt`

* **Résultat Attendu :** La sortie de strace montrera une tentative d'ouverture (openat) sur /etc/secret.txt qui échoue avec une erreur EACCES (Permission denied). De plus, une entrée DENIED correspondante apparaîtra dans les logs du noyau (sudo dmesg | grep DENIED).

Le processus de création d'un profil AppArmor transcende la simple tâche de sécurité. Il s'agit en réalité d'un puissant exercice de **"reverse engineering" comportemental**. En observant, via aa-genprof et aa-logprof, chaque fichier, chaque capacité réseau et chaque permission qu'une application tente d'utiliser, l'administrateur est forcé de comprendre son fonctionnement intime. Ce processus met en lumière le comportement réel de l'application, par opposition à son comportement attendu. Si une application, supposée être un simple outil de conversion d'images, tente d'accéder à /home/user/.ssh/id\_rsa ou d'ouvrir un socket réseau vers une destination inconnue, AppArmor le révélera immédiatement lors de la phase de création du profil. Ainsi, la création d'un profil AppArmor n'est pas seulement une mesure de confinement ; c'est aussi un **audit de comportement approfondi** qui peut démasquer des fonctionnalités inattendues, des configurations erronées, ou même des activités malveillantes cachées au sein d'une application que l'on croyait légitime.

## 

## **Section 4 : Surveillance de l'Intégrité du Système de Fichiers avec AIDE**

Cette section explique comment mettre en place un système de détection d'intrusion basé sur l'hôte (HIDS) pour surveiller les modifications non autorisées des fichiers système, des configurations et des binaires. AIDE (Advanced Intrusion Detection Environment) est l'outil de référence pour cette tâche, agissant comme un gardien silencieux qui vérifie l'intégrité du système de fichiers.

### **4.1. Installation et Initialisation de la Base de Données AIDE**

Le principe de fonctionnement d'AIDE est simple mais puissant. Il crée une "baseline" (une base de données de référence) contenant les signatures cryptographiques (telles que les hachages SHA256), les permissions, les horodatages et d'autres attributs de chaque fichier important du système. Cette baseline représente l'état "connu et bon" du serveur. Les scans ultérieurs sont ensuite comparés à cette baseline pour détecter la moindre modification, ajout ou suppression de fichier.

* **Installation :** L'installation se fait via le gestionnaire de paquets standard.  
  `sudo apt-get install aide aide-common`

Le paquet aide-common inclut des scripts utiles pour l'automatisation.

* **Initialisation :** La première étape après l'installation est de générer la baseline initiale.

`sudo aideinit`

Cette commande lit la configuration par défaut dans /etc/aide/aide.conf (qui définit les répertoires et les fichiers à surveiller) et scanne le système. Le résultat est une nouvelle base de données compressée, créée à l'emplacement /var/lib/aide/aide.db.new. Ce processus peut prendre plusieurs minutes sur un système avec de nombreux fichiers.

* **Déploiement de la Baseline :** Une fois la base de données générée, elle doit être renommée pour devenir la base de données active qu'AIDE utilisera pour les comparaisons futures.

`sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db`

Cette action doit être effectuée sur un système fraîchement installé et configuré, avant qu'il ne soit exposé à des réseaux non fiables.

* **Sécurisation de la Baseline :** C'est l'étape la plus critique du processus. Si un attaquant parvient à modifier la base de données de référence (aide.db) et le fichier de configuration (aide.conf), il peut y insérer les signatures de ses propres portes dérobées, rendant AIDE complètement aveugle à sa présence. Il est donc impératif de copier ces deux fichiers sur un support externe en lecture seule (comme un CD-R) ou de les transférer vers un serveur de stockage distant hautement sécurisé. La version de la base de données utilisée pour les vérifications doit toujours provenir de cette source sécurisée.

### **4.2. Automatisation et Analyse des Rapports d'Intégrité**

La surveillance de l'intégrité n'est efficace que si elle est effectuée régulièrement et automatiquement.

* **Automatisation :** Le paquet aide-common installe un script cron dans /etc/cron.daily/aide qui exécute une vérification quotidienne. Ce script compare l'état actuel du système à la base de données /var/lib/aide/aide.db et envoie un rapport par e-mail à l'utilisateur root si des changements sont détectés. Il est essentiel de s'assurer que les e-mails de root sont correctement redirigés vers une boîte de réception surveillée par les administrateurs.

* **Mise à Jour de la Baseline après des Changements Légitimes :** Après chaque mise à jour du système (sudo apt upgrade) ou chaque modification de configuration légitime, AIDE générera un rapport listant des dizaines, voire des centaines de fichiers modifiés. Il est alors nécessaire de mettre à jour la baseline pour refléter ce nouvel état "connu et bon". La procédure correcte est la suivante :

  1. Exécuter la commande de mise à jour : sudo aide \--update.  
  2. Cette commande crée une nouvelle base de données, aide.db.new, basée sur l'état actuel du système.  
  3. **Analyser attentivement** le rapport de différences généré entre l'ancienne et la nouvelle base de données pour s'assurer que tous les changements sont bien légitimes et correspondent aux mises à jour ou configurations effectuées.  
  4. Si tout est en ordre, remplacer l'ancienne baseline par la nouvelle : sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db.  
  5. N'oubliez pas de copier également cette nouvelle baseline sécurisée sur votre support externe.


* **Analyse des Rapports :** Un rapport AIDE typique, reçu par e-mail, ressemblera à ceci :  
  `AIDE found differences between database and filesystem!!`  
  `Start timestamp: 2023-10-27 10:00:00`

  `Summary:`  
    `Total number of files:      150000`  
    `Added files:                1`  
    `Removed files:              0`  
    `Changed files:              2`

  `---------------------------------------------------`  
  `Added files:`  
  `---------------------------------------------------`  
  `f++++++++++++++++: /etc/new-config.txt`

  `---------------------------------------------------`  
  `Changed files:`  
  `---------------------------------------------------`  
  `f   p.o..s.c..   : /etc/passwd`  
  `f .....s.c..   : /bin/ls`

Ce rapport indique qu'un nouveau fichier a été ajouté (/etc/new-config.txt) et que deux fichiers ont été modifiés. Les lettres (p.o..s.c..) indiquent quels attributs ont changé (p: permissions, s: taille, c: checksum, etc.). Une modification du checksum de /bin/ls est particulièrement suspecte et nécessite une investigation immédiate.

**Scénario de Test Pratique (AIDE)**

* **Configuration :** Sur le serveur durci, assurez-vous qu'AIDE est initialisé (sudo aideinit et sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db).

* **Action :** Simulez une modification non autorisée d'un fichier système critique.  
  `# Sur le serveur durci`  
  `sudo echo "hacker::0:0:hacker:/root:/bin/bash" >> /etc/passwd`

* **Résultat Attendu :** Lancez une vérification manuelle d'AIDE.  
  `sudo aide --check`

Le rapport généré par AIDE doit clairement indiquer que le fichier /etc/passwd a été modifié, en signalant les changements de taille, de somme de contrôle (checksum), et de temps de modification.

## 

## **Section 5 : Détection d'Intrusion Réseau (NIDS)**

Alors qu'AIDE surveille le système de l'intérieur (HIDS), un système de détection d'intrusion réseau (NIDS) agit comme un garde-frontière, inspectant le trafic réseau qui entre et sort du serveur. Il recherche des signatures d'attaques connues, des anomalies de protocole et des comportements suspects. Cette section couvre le déploiement de deux des NIDS open-source les plus réputés : Suricata et Snort.

### **5.1. Déploiement et Configuration de Suricata**

Suricata est un moteur de détection d'intrusion moderne, reconnu pour ses hautes performances grâce à son architecture multi-threadée. Il peut fonctionner en tant que NIDS (détection) ou NIPS (prévention).

* **Installation via PPA :** Pour bénéficier de la version la plus récente et stable, l'utilisation du PPA (Personal Package Archive) maintenu par les développeurs est la méthode recommandée.  
  `sudo add-apt-repository ppa:oisf/suricata-stable`  
  `sudo apt update`  
  `sudo apt install suricata`

Cette méthode garantit que vous disposez des dernières fonctionnalités et optimisations.

* **Configuration de suricata.yaml :** Le fichier de configuration principal est /etc/suricata/suricata.yaml. Les paramètres les plus importants à ajuster sont :

  * **HOME\_NET :** C'est la variable la plus critique. Elle doit être définie pour correspondre précisément aux sous-réseaux que vous considérez comme "internes" ou "protégés". De nombreuses règles de détection utilisent cette variable pour distinguer une attaque provenant de l'extérieur d'un trafic légitime à l'intérieur du réseau.  
    `HOME_NET: "[192.168.1.0/24, 10.0.0.0/8]"`

  * **EXTERNAL\_NET :** Ce paramètre est généralement défini comme "tout ce qui n'est pas HOME\_NET".  
    `EXTERNAL_NET: "!$HOME_NET"`

  * **Interface de capture (af-packet) :** Vous devez spécifier sur quelle interface réseau Suricata doit écouter.  
    `af-packet:`  
      `- interface: eth0`

Ces configurations sont essentielles pour le bon fonctionnement du moteur de détection.

* **Gestion des Règles avec suricata-update :** Suricata est aussi puissant que les règles qu'il utilise. L'outil suricata-update est le moyen standard pour télécharger, gérer et mettre à jour les ensembles de règles.  
  `sudo suricata-update`  
*   
  Par défaut, cette commande télécharge l'ensemble de règles gratuit "Emerging Threats ET Open", qui offre une excellente couverture de base. L'outil peut être configuré pour utiliser d'autres sources de règles, gratuites ou payantes.

* **Exécution et Test :**

  1. Avant de démarrer le service, il est prudent de valider la syntaxe du fichier de configuration :  
     `sudo suricata -T -c /etc/suricata/suricata.yaml -v`

    
     Cette commande testera la configuration et signalera toute erreur.


  2. Démarrez et activez le service pour qu'il se lance au démarrage :  
     `sudo systemctl start suricata`  
     `sudo systemctl enable suricata`

  3. Les alertes sont généralement enregistrées dans deux fichiers principaux dans /var/log/suricata/ :


     * fast.log : Un format texte simple, lisible par l'homme, contenant une ligne par alerte.  
     * eve.json : Un format JSON beaucoup plus riche, contenant non seulement les alertes mais aussi les métadonnées de flux réseau, les logs DNS, HTTP, etc. C'est le format à privilégier pour l'intégration avec des outils comme un SIEM ou la suite Elastic.

### 

### **5.2. Alternative : Déploiement de Snort 3**

Snort est le NIDS open-source pionnier, aujourd'hui soutenu par Cisco Talos. Snort 3 est une réécriture majeure qui modernise son architecture et sa configuration.

* **Installation depuis les Sources :** La version 3 de Snort n'est souvent pas disponible dans les dépôts standards d'Ubuntu. L'installation depuis les sources est donc la méthode privilégiée pour obtenir la dernière version.

  1. Installer les dépendances de compilation.  
  2. Télécharger et compiler libdaq (Data Acquisition library), une dépendance requise par Snort 3\.  
  3. Télécharger les sources de Snort 3, configurer, compiler et installer.


* **Configuration de snort.lua :** Snort 3 abandonne l'ancien snort.conf au profit d'une configuration plus flexible et puissante basée sur le langage de script Lua.

  * Les variables réseau sont définies dans /usr/local/etc/snort/snort\_defaults.lua ou directement dans le fichier de configuration principal, /usr/local/etc/snort/snort.lua.

    
    `HOME_NET = '[192.168.1.0/24]'`  
    `EXTERNAL_NET = '!$HOME_NET'`

La logique reste la même que pour Suricata : définir correctement le réseau à protéger est fondamental.

* **Gestion des Règles :** Les règles pour Snort peuvent être téléchargées depuis le site officiel Snort.org. Il existe un ensemble de règles communautaires (gratuites) et des règles d'abonnés (payantes, avec un accès plus rapide aux nouvelles signatures). Ces règles doivent être placées dans le répertoire approprié et incluses dans le fichier snort.lua.

* **Exécution et Test :**  
  1. Valider la configuration :  
     `snort -T -c /usr/local/etc/snort/snort.lua`  
     Cette commande vérifie la syntaxe de la configuration Lua et des règles chargées.  
  2. Lancer Snort en mode NIDS pour écouter sur une interface :  
     `sudo snort -q -A console -i eth0 -c /usr/local/etc/snort/snort.lua`  
     L'option \-A console affiche les alertes sur la sortie standard, ce qui est utile pour les tests. Pour une utilisation en production, on configure généralement la sortie vers des fichiers journaux ou un socket syslog.


  
**Scénario de Test Pratique (NIDS)**

* **Configuration :** Un NIDS (Suricata ou Snort) est en cours d'exécution sur le serveur durci ("Victime").

* **Action (sur l'attaquant) :** Utilisez nmap pour effectuer un scan de ports de base sur la victime. Ce type de scan est une activité de reconnaissance typique qu'un NIDS devrait détecter.  
  `nmap -sS 192.168.1.100`

Pour un test plus direct, générez une alerte de test. Créez une règle personnalisée dans Suricata (/etc/suricata/rules/local.rules) qui se déclenche sur un simple ping : alert icmp any any \-\> $HOME\_NET any (msg:"ICMP Test Rule"; sid:1000001; rev:1;). Rechargez les règles. Puis, depuis l'attaquant, envoyez un ping : ping 192.168.1.100.

* **Résultat Attendu :** Sur la victime, consultez les logs d'alertes (tail \-f /var/log/suricata/fast.log). Une ou plusieurs alertes correspondant au scan Nmap ou au ping de test devraient apparaître, confirmant que le NIDS inspecte activement le trafic et que les règles sont fonctionnelles.

## 

## **Section 6 : Audit de Sécurité et Détection de Rootkits**

Après avoir appliqué les mesures de durcissement des sections précédentes, il est essentiel de disposer d'outils pour vérifier leur efficacité, auditer la conformité du système par rapport aux bonnes pratiques, et scanner régulièrement le serveur à la recherche de logiciels malveillants qui auraient pu déjouer les défenses en place.

### **6.1. Audit de Conformité Continu avec Lynis**

Lynis est un outil d'audit de sécurité open-source qui effectue un scan approfondi du système. Il ne modifie aucune configuration, mais compare l'état actuel du serveur à une vaste base de connaissances de bonnes pratiques de sécurité et de durcissement. Son rapport final fournit un score de durcissement et une liste de suggestions concrètes pour améliorer la posture de sécurité.

* **Installation via le Dépôt Cisofy :** Pour garantir l'utilisation de la version la plus récente, qui contient les derniers tests de sécurité, il est fortement recommandé d'utiliser le dépôt officiel de l'éditeur.  
  `wget -O - https://packages.cisofy.com/keys/cisofy-software-public.key | sudo apt-key add -`  
  `echo "deb https://packages.cisofy.com/community/lynis/deb/ stable main" | sudo tee /etc/apt/sources.list.d/cisofy-lynis.list`  
  `sudo apt update`  
  `sudo apt install lynis`

Cette méthode est plus fiable que l'installation depuis les dépôts Ubuntu, qui peuvent contenir une version plus ancienne.

* **Exécution de l'Audit :** L'audit complet du système se lance avec une seule commande. Il est préférable de l'exécuter avec des privilèges root pour permettre à Lynis d'inspecter tous les aspects du système.  
  `sudo lynis audit system`

Le scan est interactif par défaut, faisant une pause après chaque grande section. Pour une exécution non interactive (par exemple dans un script), on peut utiliser l'option \--quick.

* **Analyse du Rapport :** À la fin du scan, Lynis affiche un résumé directement sur la console, incluant le score de durcissement et la localisation des fichiers de rapport.

  * Le rapport complet et détaillé se trouve dans /var/log/lynis.log.  
  * Un fichier de données plus structuré est disponible dans /var/log/lynis-report.dat.  
  * La section la plus importante de la sortie est la liste des **Suggestions**. Chaque suggestion est une recommandation d'action pour corriger une faiblesse de configuration identifiée.  
  * Chaque test effectué par Lynis est associé à un identifiant unique (par exemple, KRNL-5820). Pour obtenir des informations détaillées sur un test spécifique et la justification de la suggestion, on peut utiliser la commande :  
    `lynis show details KRNL-5820`

L'objectif est de traiter systématiquement chaque WARNING et chaque SUGGESTION pour améliorer progressivement le score de durcissement du serveur.

### **6.2. Chasse aux Rootkits avec rkhunter et chkrootkit**

Les rootkits sont des logiciels malveillants conçus pour se cacher profondément dans le système d'exploitation, souvent en modifiant des binaires système ou des modules du noyau pour dissimuler leur présence. rkhunter (Rootkit Hunter) et chkrootkit sont deux outils spécialisés dans la détection de ces menaces.

* **Installation :** Les deux outils sont disponibles dans les dépôts standards d'Ubuntu.  
  `sudo apt install rkhunter chkrootkit`

* **rkhunter :**  
  * **Initialisation et Mise à Jour :** rkhunter fonctionne en comparant les propriétés actuelles des fichiers système à une baseline. Il est crucial de comprendre la différence entre ses deux commandes de mise à jour :


    1. sudo rkhunter \--propupd : Cette commande crée ou met à jour la **baseline locale** des propriétés des fichiers (hachages, tailles, permissions). Elle doit être exécutée pour la première fois sur un système connu pour être sain, puis après chaque mise à jour légitime du système pour éviter les faux positifs.

    

    2. sudo rkhunter \--update : Cette commande télécharge les **mises à jour des définitions de menaces** (signatures de rootkits connus, listes de fichiers suspects) depuis Internet.

    

  * **Scan :** Pour lancer un scan complet du système :  
    `sudo rkhunter --check`

  * **Gestion des Faux Positifs :** rkhunter est connu pour générer des avertissements, notamment après des mises à jour de paquets. Si une alerte est identifiée comme un faux positif, elle peut être "whitelisted" (mise en liste blanche) dans le fichier de configuration /etc/rkhunter.conf pour qu'elle ne soit plus signalée lors des scans futurs.


* **chkrootkit :**

  * **Utilisation :** chkrootkit est plus simple à utiliser. Il ne maintient pas de baseline de la même manière que rkhunter. Il effectue une série de tests directs pour des signatures de rootkits connus, des modifications de binaires et des anomalies système.  
    `sudo chkrootkit`

Il est plus rapide mais potentiellement moins complet que rkhunter.

**Scénario de Test Pratique (rkhunter)**

* **Configuration :** Sur le serveur durci, exécutez sudo rkhunter \--propupd pour créer une baseline saine.

* **Action :** Simulez la modification d'un binaire système, une technique couramment utilisée par les rootkits.  
  `# Sur le serveur durci`  
  `sudo touch /bin/ls`

* **Résultat Attendu :** Lancez un scan rkhunter.  
  `sudo rkhunter --check`  
  Le rapport de rkhunter devrait maintenant afficher un \`\` pour le fichier /bin/ls, indiquant que ses propriétés (dans ce cas, l'horodatage de modification) ont changé par rapport à la baseline stockée. Cela confirme que le mécanisme de détection de rkhunter est opérationnel.

Il est important de ne pas considérer rkhunter et chkrootkit comme des concurrents, mais plutôt comme des outils **complémentaires**. rkhunter est plus exhaustif avec sa vérification des propriétés de fichiers et ses définitions de menaces mises à jour, tandis que chkrootkit est plus rapide et effectue certains tests spécifiques, comme la recherche d'interfaces réseau en mode promiscuous, que rkhunter n'aborde pas de la même manière. La meilleure pratique consiste à planifier l'exécution des deux outils, de manière séquentielle (jamais simultanément pour éviter les interférences et les faux positifs). Il faut également reconnaître leur principale limite : ils sont principalement basés sur des signatures et des heuristiques connues. Un rootkit moderne et sophistiqué ("zero-day") pourrait ne pas être détecté. C'est pourquoi leur utilisation doit impérativement être combinée avec des outils de surveillance comportementale comme AppArmor (qui peut empêcher un rootkit de s'exécuter) et des outils de surveillance de l'intégrité comme AIDE (qui peut détecter les modifications de fichiers causées par le rootkit).

## 

## **Conclusion**

### **Synthèse des Couches de Défense**

Ce guide a détaillé la construction d'une forteresse numérique autour d'un serveur Ubuntu 20.04, en appliquant le principe de défense en profondeur. Chaque section a ajouté une couche de protection stratégique, créant un système où les défenses se renforcent mutuellement.

* **Le Noyau et le Système** ont été durcis à la base, en modifiant les paramètres sysctl pour repousser les attaques réseau de bas niveau et en réduisant la surface d'attaque en désactivant les modules inutiles.

* **Le Système de Fichiers** a été verrouillé par un partitionnement intelligent et des options de montage restrictives, transformant des zones potentiellement dangereuses comme /tmp et /dev/shm en simples espaces de stockage de données inertes.

* **Les Applications** ont été confinées avec AppArmor, les limitant à leurs fonctions strictement nécessaires et empêchant qu'une vulnérabilité dans un service ne se propage à l'ensemble du système.

* **L'Intégrité** du système est désormais sous surveillance constante grâce à AIDE, qui agit comme un système d'alarme silencieux, signalant toute modification de fichier non autorisée.

* **Le Réseau** est inspecté par un NIDS (Suricata ou Snort), qui analyse le trafic à la recherche de signatures d'attaques connues et de comportements anormaux.

* **L'Ensemble du Système** est audité régulièrement avec Lynis pour assurer la conformité aux bonnes pratiques et scanné avec rkhunter et chkrootkit pour chasser les logiciels malveillants qui auraient pu s'infiltrer.

### **La Sécurité comme Processus Continu**

Il est crucial de comprendre que le durcissement d'un système n'est pas une action ponctuelle, mais le début d'un processus continu. La sécurité informatique est un cycle dynamique qui exige une vigilance constante. Les menaces évoluent, de nouvelles vulnérabilités sont découvertes et les configurations peuvent dériver avec le temps. La maintenance d'une posture de sécurité robuste repose sur un cycle vertueux : la **mise à jour** régulière des systèmes d'exploitation, des applications et des règles de détection ; la **surveillance** proactive des journaux et des alertes générés par les outils mis en place ; et l'**adaptation** continue des défenses pour contrer les nouvelles menaces et les nouvelles techniques d'attaque. Un serveur sécurisé aujourd'hui ne le restera demain que grâce à l'engagement continu de ses administrateurs dans ce processus de sécurité itératif.

#### 

#### **Sources des citations**

1\. List of things for hardening Ubuntu \- GitHub Gist, https://gist.github.com/lokhman/cc716d2e2d373dd696b2d9264c0287a3 2\. Sysctl Hardening: Advanced Linux Kernel Security Techniques \- CalCom Software, https://calcomsoftware.com/sysctl-configuration-hardening/ 3\. Firewall Testing with Hping3: A Comprehensive Guide \- Infosec Train, https://www.infosectrain.com/blog/firewall-testing-with-hping3-a-comprehensive-guide/ 4\. rp\_filter | sysctl-explorer.net, https://sysctl-explorer.net/net/ipv4/rp\_filter/ 5\. 13.1. Reverse Path Filtering, https://tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.kernel.rpf.html 6\. Hping3 & Traffic/Attack Generator \- NCCS, https://nccs.gov.in/public/events/DDoS\_Presentation\_17092024.pdf 7\. TCP SYN Queue and Accept Queue Overflow Explained \- Alibaba Cloud Community, https://www.alibabacloud.com/blog/599203 8\. Guide to the Secure Configuration of Ubuntu 20.04 \- Static OpenSCAP, http://static.open-scap.org/ssg-guides/ssg-ubuntu2004-guide-cis\_level1\_server.html 9\. ICMP Redirect Attack. Objective: | by Katz \- Medium, https://medium.com/@sherishrat/icmp-redirect-attack-18755cd0897 10\. konstruktoid/hardening: Hardening Ubuntu. Systemd edition. \- GitHub, https://github.com/konstruktoid/hardening 11\. How to Secure /tmp and /var/tmp and /dev/shm on Linux \- Skynats, https://www.skynats.com/blog/secure-tmp-and-var-tmp-and-dev-shm-on-linux/ 12\. Securing /tmp on a linux server \- ITsyndicate, https://itsyndicate.org/blog/securing-tmp-on-a-linux-server/ 13\. Securing mount points on Linux, https://linux-audit.com/securing-mount-points-on-linux/ 14\. hping3 | Kali Linux Tools, https://www.kali.org/tools/hping3/ 15\. What is /dev/shm and how to mount /dev/shm \- Sharad Chhetri, https://sharadchhetri.com/devshm-mount-devshm 16\. Ensure /dev/shm is Mounted on a Separate Partition for Secure Shared Memory \- iCompaas Support, https://support.icompaas.com/support/solutions/articles/62000234966-ensure-dev-shm-is-mounted-on-a-separate-partition-for-secure-shared-memory 17\. How to Use AppArmor to Secure Applications in Ubuntu \- ANOVIN, https://anovin.mk/tutorial/how-to-use-apparmor-to-secure-applications-in-ubuntu/ 18\. How to Configure AppArmor for Security on Debian or Ubuntu \- JumpCloud, https://jumpcloud.com/blog/how-to-configure-apparmor-for-security-debian-ubuntu 19\. AppArmor \- Ubuntu Server documentation, https://documentation.ubuntu.com/server/how-to/security/apparmor/ 20\. AppArmor \- Ubuntu security documentation, https://documentation.ubuntu.com/security/docs/security-features/privilege-restriction/apparmor/ 21\. AppArmor/HowToUse \- Debian Wiki, https://wiki.debian.org/AppArmor/HowToUse 22\. Build and Test AIDE Database \- Datadog Docs, https://docs.datadoghq.com/security/default\_rules/def-000-qbv/ 23\. How to Check Integrity of File and Directory Using "AIDE" in Linux, https://www.tecmint.com/check-integrity-of-file-and-directory-using-aide-in-linux/ 24\. AIDE File Integrity Safeguarding Your Linux System \- Johnson Premier Blog, https://blog.johnsonpremier.net/aide\_file\_integrity/ 25\. Install and Setup Suricata on Ubuntu 22.04/Ubuntu 20.04 \- kifarunix.com, https://kifarunix.com/install-and-setup-suricata-on-ubuntu-22-04-ubuntu-20-04/ 26\. How to install Suricata on Ubuntu to secure your network \- Hostinger, https://www.hostinger.com/tutorials/how-to-install-suricata-on-ubuntu 27\. How To Install Suricata on Ubuntu 20.04 | DigitalOcean, https://www.digitalocean.com/community/tutorials/how-to-install-suricata-on-ubuntu-20-04 28\. How to Install And Setup Suricata IDS on Ubuntu | Atlantic.Net, https://www.atlantic.net/vps-hosting/how-to-install-and-setup-suricata-ids-on-ubuntu/ 29\. How To Configure Suricata as an Intrusion Prevention System (IPS) on Ubuntu 20.04, https://www.digitalocean.com/community/tutorials/how-to-configure-suricata-as-an-intrusion-prevention-system-ips-on-ubuntu-20-04 30\. Configuring a Network Intrusion Detection System with Snort on Ubuntu 20.04 \- Reintech, https://reintech.io/blog/configure-snort-network-intrusion-detection-ubuntu 31\. Snort \- Network Intrusion Detection & Prevention System, https://www.snort.org/ 32\. Install and Configure Snort 3 NIDS on Ubuntu 20.04 \- kifarunix.com, https://kifarunix.com/install-and-configure-snort-3-nids-on-ubuntu-20-04/ 33\. How to Install Snort on Ubuntu 20.04 \- LinuxOPsys, https://linuxopsys.com/install-snort-on-ubuntu 34\. How to Install Snort NIDS on Ubuntu Linux | Rapid7 Blog, https://www.rapid7.com/blog/post/2017/01/11/how-to-install-snort-nids-on-ubuntu-linux/ 35\. Snort Tutorial and Practical Examples \- HackerTarget.com, https://hackertarget.com/snort-tutorial-practical-examples/ 36\. NMAP Vulnerability Scan: How To Easily Run And Assess Risk \- Datamation, https://www.datamation.com/security/how-to-easily-run-a-vulnerability-scan-using-nmap/ 37\. Information Security Testing and Auditing with Nmap \- Pluralsight, https://www.pluralsight.com/paths/information-security-testing-and-auditing-with-nmap 38\. Install and Setup Lynis Security Auditing tool on Ubuntu 20.04 ..., https://kifarunix.com/install-and-setup-lynis-security-auditing-tool-on-ubuntu-20-04/ 39\. Get Started with Lynis \- Installation Guide \- CISOfy, https://cisofy.com/documentation/lynis/get-started/ 40\. Lynis Installation and Usage Guide \- CISOfy, https://cisofy.com/documentation/lynis/ 41\. Detecting and Checking Rootkits with Chkrootkit and rkhunter Tool ..., https://www.geeksforgeeks.org/linux-unix/detecting-and-checking-rootkits-with-chkrootkit-and-rkhunter-tool-in-kali-linux/ 42\. Rootkit Detection — chkrootkit & rkhunter \- Medium, https://medium.com/@rkone1552000/rootkit-detection-chkrootkit-rkhunter-f52394116861 43\. propupd \- Rootkit Hunter Wiki \- SourceForge, https://sourceforge.net/p/rkhunter/wiki/propupd/ 44\. RKhunter \- Community Help Wiki, https://help.ubuntu.com/community/RKhunter 45\. How to Install and Use Rkhunter on Ubuntu 22.04 & 20.04 \- TecAdmin, https://tecadmin.net/how-to-install-rkhunter-on-ubuntu/ 46\. Rootkit Hunter and ChkRootKit \- Tutorials & Guides \- Zorin Forum, https://forum.zorin.com/t/rootkit-hunter-and-chkrootkit/25564 47\. Is there any conflicts between running rkhunter and chkrootkit on one system? \- Ask Ubuntu, https://askubuntu.com/questions/724885/is-there-any-conflicts-between-running-rkhunter-and-chkrootkit-on-one-system