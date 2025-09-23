
# Déploiement d'un WAF ModSecurity de niveau production avec Apache sur Ubuntu 24.04 : Un guide d'implémentation complet et une analyse critique

## Partie I : Le guide de laboratoire définitif pour le déploiement de ModSecurity et de l'OWASP CRS

Ce guide fournit une méthodologie complète et éprouvée pour la mise en place d'un pare-feu applicatif web robuste basé sur ModSecurity et l'OWASP Core Rule Set sur un serveur web Apache fonctionnant sous Ubuntu 24.04. Chaque étape est conçue pour construire une couche de défense sécurisée, maintenable et prête pour la production.

### 1.0 Section 1 : Préparation de l'environnement de base

Avant de déployer des contrôles de sécurité avancés, il est impératif de s'assurer que le système d'exploitation et les services sous-jacents sont correctement configurés et sécurisés.

#### 1.1 Provisionnement du système et sécurité de base sur Ubuntu 24.04

La procédure commence avec une instance de serveur Ubuntu 24.04 nouvellement déployée.5 La première action, une pratique d'hygiène fondamentale en administration système, consiste à mettre à jour l'index des paquets et à appliquer toutes les mises à jour de sécurité et de logiciels disponibles.

`sudo apt update && sudo apt upgrade -y`

Ensuite, pour adhérer au principe du moindre privilège, toutes les opérations suivantes doivent être effectuées par un utilisateur non-root disposant de privilèges sudo. L'utilisation directe du compte root pour les tâches administratives quotidiennes est une pratique à haut risque qui devrait être évitée.5

#### 1.2 Installation et gestion du service Apache2

Le serveur web Apache est disponible dans les dépôts par défaut d'Ubuntu 24.04, ce qui garantit une installation simple et une maintenance facilitée.5

`sudo apt install apache2 -y`

Une fois l'installation terminée, il est judicieux de vérifier la version installée et l'état du service.

#### Vérifier la version d'Apache
`apache2 -v`

#### Vérifier l'état du service Apache
`sudo systemctl status apache2`


Le service Apache doit être configuré pour démarrer automatiquement au démarrage du système afin d'assurer la disponibilité du service web après un redémarrage.

`sudo systemctl enable apache2`

Les commandes de base pour gérer le service (start, stop, restart) sont essentielles pour l'administration continue du serveur.5

#### 1.3 Configuration du pare-feu avec UFW pour les services web

La configuration d'un pare-feu au niveau de l'hôte est une étape de sécurité non négociable. UFW (Uncomplicated Firewall) fournit une interface conviviale pour gérer iptables sur Ubuntu.

##### Lister les profils d'application disponibles
`sudo ufw app list`

La sortie de cette commande affichera des profils pour Apache, notamment 'Apache Full', qui autorise le trafic sur les ports 80 (HTTP) et 443 (HTTPS).5

##### Autoriser le trafic web standard
`sudo ufw allow 'Apache Full'`

##### Activer le pare-feu
`sudo ufw enable`

##### Vérifier l'état et les règles actives
`sudo ufw status`


Cette configuration garantit que seules les connexions SSH (généralement sur le port 22) et web (ports 80 et 443) sont autorisées à entrer sur le serveur, bloquant tout autre trafic non sollicité.

### 2.0 Section 2 : Installation et configuration du moteur ModSecurity

Cette section se concentre exclusivement sur l'installation et la configuration du module ModSecurity pour Apache, qui constitue le moteur d'inspection du WAF.

#### 2.1 Déploiement du module libapache2-mod-security2

Le module ModSecurity est facilement installable depuis les dépôts officiels d'Ubuntu, ce qui simplifie grandement le processus de déploiement.2


`sudo apt install libapache2-mod-security2 -y`


Le processus d'installation du paquet active généralement le module Apache automatiquement. Il est néanmoins prudent de le vérifier et de l'activer manuellement si nécessaire.2


`sudo a2enmod security2`


Après l'activation d'un nouveau module, un redémarrage d'Apache est requis pour qu'il soit chargé en mémoire.


`sudo systemctl restart apache2`


### 2.2 Établissement de la configuration de base modsecurity.conf

Le paquet d'installation fournit un fichier de configuration modèle. La meilleure pratique consiste à copier ce fichier pour créer la configuration active, ce qui permet de conserver l'original comme référence.2

`sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf`


Ce fichier, /etc/modsecurity/modsecurity.conf, devient le fichier de configuration principal pour le moteur ModSecurity lui-même. Il contrôle le comportement global du WAF, indépendamment des règles spécifiques qui seront chargées par la suite.

#### 2.3 Examen approfondi des directives essentielles

Plusieurs directives dans modsecurity.conf sont d'une importance capitale et doivent être comprises et configurées correctement.
- SecRuleEngine: C'est l'interrupteur principal du WAF. Il peut prendre trois valeurs : On (les règles sont traitées et les actions perturbatrices comme deny sont exécutées), Off (le module est complètement désactivé), et DetectionOnly (les règles sont traitées et les correspondances sont journalisées, mais aucune action perturbatrice n'est effectuée).11 Pour une nouvelle installation, il est crucial de commencer en mode
- DetectionOnly. Cette approche permet d'observer le comportement du WAF et d'identifier les faux positifs (blocages de trafic légitime) sans affecter les utilisateurs. Le passage en mode On ne doit se faire qu'après une période de surveillance et d'ajustement.
- SecAuditEngine: Cette directive contrôle le moteur de journalisation d'audit. La valeur On garantit que toutes les transactions traitées par ModSecurity sont enregistrées dans le journal d'audit, fournissant une trace complète pour l'analyse et le débogage.3 L'alternative
- RelevantOnly ne journalise que les transactions qui ont déclenché une alerte ou une erreur, ce qui peut être utile dans des environnements à très fort trafic pour réduire le volume des journaux, mais On est recommandé pour la phase initiale de déploiement et de réglage.12
- SecAuditLog: Définit le chemin d'accès au fichier journal d'audit détaillé. Sur les systèmes basés sur Debian/Ubuntu, ce chemin est généralement /var/log/apache2/modsec_audit.log.3 Ce fichier est la source principale d'informations pour l'analyse forensique des requêtes bloquées.
- SecRequestBodyAccess On: Cette directive est absolument critique. Par défaut, pour des raisons de performance, ModSecurity n'inspecte pas le corps des requêtes (par exemple, les données envoyées via un formulaire POST). Définir SecRequestBodyAccess sur On ordonne à ModSecurity de mettre en mémoire tampon et d'analyser le corps des requêtes. Sans cette directive, le WAF est aveugle à une grande majorité des attaques applicatives, telles que les injections SQL ou XSS soumises via des formulaires web.
- SecRequestBodyLimit: Cette directive, ainsi que d'autres directives de limites similaires (SecRequestBodyNoFilesLimit, etc.), est une mesure de défense contre les attaques par déni de service (DoS). Elle définit la taille maximale du corps d'une requête que ModSecurity traitera. Les requêtes dépassant cette limite seront rejetées, empêchant un attaquant de saturer le serveur avec des requêtes volumineuses.12

### 3.0 Section 3 : Intégration de la dernière version de l'OWASP Core Rule Set (CRS)

Cette section est la plus importante du guide. Elle détaille l'intégration de l'intelligence qui alimente le moteur ModSecurity, une étape entièrement omise par le tutoriel initialement analysé.

#### 3.1 Le rôle du CRS en tant que couche d'intelligence du WAF

Comme établi précédemment, ModSecurity est le moteur, mais le CRS est le "cerveau". C'est cet ensemble de règles qui contient les signatures et les heuristiques nécessaires pour détecter les tentatives d'attaques connues, des injections SQL aux scripts intersites, en passant par les inclusions de fichiers et bien d'autres.8

#### 3.2 Acquisition et structuration de la dernière version stable du CRS

Il ne suffit pas de "télécharger les règles". La source, la version et la structure de l'installation sont des facteurs critiques pour la sécurité et la maintenabilité. Le paysage des menaces évolue constamment, et l'utilisation d'une version obsolète ou non supportée du CRS expose le serveur à de nouvelles vulnérabilités et techniques de contournement. Par exemple, le projet a migré de son ancien dépôt chez SpiderLabs vers une organisation GitHub dédiée, coreruleset.15 L'utilisation de l'ancienne source est une erreur critique. De plus, l'équipe du CRS publie régulièrement des mises à jour de sécurité ; une version récente a corrigé une vulnérabilité de contournement du corps de la requête.16 L'utilisation d'une version non supportée (par exemple, la version 3.2.x est obsolète selon la politique de support 17) laisse le WAF vulnérable.
La procédure correcte consiste à se rendre sur la page officielle des publications du projet, à identifier la dernière version stable majeure, et à télécharger son archive source.
Télécharger la dernière version stable :
Note : Au moment de la rédaction de ce guide, la version 4.18.0 est la dernière version stable. Il est impératif de vérifier la page des publications sur https://github.com/coreruleset/coreruleset/releases pour la version la plus récente. 9

`cd /tmp`
`wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v4.18.0.tar.gz`


Extraire et organiser les fichiers :

`tar -xzvf v4.18.0.tar.gz`
`sudo mv coreruleset-4.18.0 /etc/apache2/modsecurity-crs`

Cette dernière commande déplace les règles dans un répertoire dédié et clairement identifié, /etc/apache2/modsecurity-crs, ce qui sépare la configuration personnalisée des fichiers système gérés par le gestionnaire de paquets et facilite les futures mises à jour.2

#### 3.3 Configuration de crs-setup.conf : niveaux de paranoïa et score d'anomalie

Le CRS est livré avec un fichier de configuration modèle qui doit être activé et personnalisé.




`cd /etc/apache2/modsecurity-crs`
`sudo cp crs-setup.conf.example crs-setup.conf`


Ce fichier, crs-setup.conf, est l'endroit où sont définis les paramètres de fonctionnement de base du CRS. Deux concepts sont fondamentaux :
Mode de score d'anomalie : Contrairement à une approche binaire où la première règle correspondante bloque la requête, le CRS utilise un système de notation. Chaque règle qui détecte une anomalie potentielle n'ajoute qu'un certain nombre de points au "score d'anomalie" de la transaction. Ce n'est que si le score total dépasse un seuil prédéfini que la requête est bloquée. Cette approche réduit considérablement les faux positifs, car une seule règle de faible gravité ne bloquera pas une requête légitime.
Niveaux de paranoïa (Paranoia Levels - PL) : Le CRS est organisé en quatre niveaux de paranoïa, offrant un compromis entre la sécurité et le risque de faux positifs. Le niveau de paranoïa est l'un des paramètres les plus importants à configurer dans crs-setup.conf.
Niveau de Paranoïa
Description
Cas d'utilisation typique
Risque de faux positifs
PL1 (Défaut)
Protection de base contre les attaques les plus courantes avec un très faible risque de faux positifs. Inclut la plupart des protections du Top 10 de l'OWASP.
Tous les sites web et API à usage général. Le point de départ recommandé.
Faible
PL2
Sécurité accrue, incluant des règles contre des attaques plus avancées (par ex., certaines variantes d'injection de commandes).
Sites de commerce électronique, applications manipulant des données sensibles. Nécessite quelques ajustements.
Moyen
PL3
Sécurité complète avec des règles plus restrictives. Peut signaler un comportement légitime mais inhabituel de l'application.
Applications à haute sécurité (par ex., gouvernement, finance, santé). Nécessite un réglage important.
Élevé
PL4
Niveau paranoïaque. Extrêmement restrictif, conçu pour une sécurité maximale dans des environnements critiques.
API ou segments d'application avec des modèles de trafic très prévisibles et restreints.
Très élevé

Pour commencer, il est recommandé de définir le niveau de paranoïa à 1 en éditant la ligne correspondante dans crs-setup.conf.

#### 3.4 Activation du CRS dans la configuration d'Apache

La dernière étape consiste à indiquer à Apache de charger les fichiers de configuration et de règles du CRS. Cela se fait dans le fichier de configuration du module ModSecurity, /etc/apache2/mods-enabled/security2.conf.
L'ordre dans lequel les fichiers sont inclus est d'une importance capitale. Le fichier de configuration crs-setup.conf définit des variables (comme le niveau de paranoïa) qui sont ensuite utilisées par les fichiers de règles dans le répertoire rules/. Si les règles sont chargées avant le fichier de configuration, elles s'exécuteront avec des paramètres par défaut, ignorant toute personnalisation. Par conséquent, l'ordre d'inclusion doit être strictement respecté.
Ajoutez les lignes suivantes à la fin de /etc/apache2/mods-enabled/security2.conf 2 :

Apache

```
<IfModule security2_module>
    #... autres directives existantes...

    # Inclure la configuration du CRS (DOIT ÊTRE EN PREMIER)
    IncludeOptional /etc/apache2/modsecurity-crs/crs-setup.conf

    # Inclure les fichiers de règles du CRS
    IncludeOptional /etc/apache2/modsecurity-crs/rules/*.conf
</IfModule>
```


### 4.0 Section 4 : Vérification du système et simulation d'attaques

Avec le moteur et les règles en place, il est temps de valider que la configuration est correcte et que le WAF fonctionne comme prévu.

#### 4.1 Vérification de la syntaxe de configuration et redémarrage du service

Avant de redémarrer un service critique comme Apache, il est impératif de vérifier la syntaxe de ses fichiers de configuration. Cela évite qu'une erreur de frappe ne mette le serveur web hors service.20

`sudo apache2ctl configtest`

Si la commande retourne Syntax OK, le service peut être redémarré en toute sécurité pour appliquer toutes les modifications.

`sudo systemctl restart apache2`

#### 4.2 Simulation de vecteurs d'attaque courants avec curl

La meilleure façon de vérifier que le WAF est actif est de lui envoyer des requêtes malveillantes simples. L'outil en ligne de commande curl est parfait pour cela. Ces tests permettent de visualiser directement l'intervention du WAF. Au lieu de recevoir une réponse 200 OK ou 404 Not Found de l'application, une requête malveillante devrait être interceptée et recevoir une réponse 403 Forbidden du WAF.
Test de "Path Traversal" : Tente de lire un fichier système.

`curl -i "http://<votre_ip_serveur>/?exec=/etc/passwd"`

Attendu : Une réponse HTTP/1.1 403 Forbidden.21
Test de Cross-Site Scripting (XSS) : Tente d'injecter une balise de script.

`curl -i "http://<votre_ip_serveur>/?search=<script>alert('xss')</script>"`

Attendu : Une réponse HTTP/1.1 403 Forbidden.20
Test d'injection SQL (SQLi) : Tente de contourner une logique de requête SQL.

`curl -i "http://<votre_ip_serveur>/?id=1' OR 1=1--"`

Attendu : Une réponse HTTP/1.1 403 Forbidden.20

#### 4.3 Analyse des réponses 403 Forbidden attendues

La réception d'un code de statut 403 Forbidden confirme que :
La requête a bien atteint le serveur Apache.
Le module ModSecurity a intercepté et analysé la requête.
Une ou plusieurs règles de l'OWASP CRS ont identifié la charge utile comme malveillante.
Le score d'anomalie de la transaction a dépassé le seuil de blocage.
ModSecurity a exécuté l'action par défaut (deny), interrompant la requête et renvoyant une réponse 403.

### 5.0 Section 5 : Maîtrise de la journalisation et de l'analyse des alertes ModSecurity

Un WAF qui bloque silencieusement est difficile à gérer. Comprendre comment lire et interpréter ses journaux est une compétence essentielle pour le réglage, la réponse aux incidents et la compréhension du paysage des menaces visant votre application.

#### 5.1 Navigation dans le journal d'erreurs d'Apache et le journal d'audit de ModSecurity

Il existe deux fichiers journaux principaux à surveiller :
Journal d'erreurs d'Apache (/var/log/apache2/error.log) : C'est le premier endroit où regarder. Pour chaque requête bloquée, ModSecurity y inscrira une entrée concise. Cette entrée est le signal : elle vous alerte qu'un événement s'est produit et fournit des informations clés comme le message de la règle déclenchée, son ID unique, et un identifiant de transaction unique pour cette requête spécifique.11

`sudo tail -f /var/log/apache2/error.log`


Journal d'audit de ModSecurity (/var/log/apache2/modsec_audit.log) : C'est le journal forensique. Il contient tous les détails imaginables sur chaque transaction traitée par ModSecurity. C'est le contexte : il fournit les preuves complètes de ce qui s'est passé.3

`sudo tail -f /var/log/apache2/modsec_audit.log`


Le flux de travail pour analyser un événement de blocage est symbiotique. Un administrateur efficace repère d'abord l'alerte dans le journal d'erreurs d'Apache, qui est plus facile à lire. Il extrait ensuite l'identifiant de transaction unique de cette alerte (par exemple, [unique_id "Zabcdefg12345"]). Enfin, il utilise cet identifiant pour rechercher (grep) l'entrée correspondante complète dans le journal d'audit de ModSecurity. Ce processus permet de passer rapidement d'une alerte de haut niveau à une analyse forensique détaillée de la requête, ce qui est fondamental pour distinguer une véritable attaque d'un faux positif.

#### 5.2 Anatomie d'une entrée du journal modsec_audit.log

Chaque entrée dans le journal d'audit est composée de plusieurs sections, chacune identifiée par une lettre. Comprendre ces sections est la clé pour une analyse efficace.

Section
Nom
Description
Référence
A
En-tête d'audit
Métadonnées de la transaction : Horodatage, ID unique, IP et port source/destination.
13
B
En-têtes de la requête
La liste complète des en-têtes de la requête HTTP envoyés par le client.
23
C
Corps de la requête
La charge utile de la requête (par ex., données POST). Crucial pour analyser les soumissions de formulaires.
23
E
Corps de la réponse
Le corps de la réponse envoyée par l'application web avant que ModSecurity ne la bloque. Utile pour l'analyse des fuites de données.
23
F
En-têtes de la réponse
Les en-têtes envoyés par l'application web.
23
H
Pied de page d'audit
Contient des informations récapitulatives, y compris les messages des règles qui ont été déclenchées.
23
K
Règles correspondantes
Une liste consolidée de toutes les règles qui ont correspondu pour la transaction. Essentiel pour un diagnostic rapide.
23
Z
Délimiteur final
Marque la fin de l'entrée du journal d'audit pour cette transaction.
13


#### 5.3 Exemple pratique : tracer une requête bloquée de la charge utile à l'ID de la règle

Supposons que nous exécutions le test d'injection SQL précédent : curl -i "http://<votre_ip_serveur>/?id=1' OR 1=1--".
Dans /var/log/apache2/error.log, une ligne similaire à celle-ci apparaîtra :
[modsecurity]... [id "942100"]... [unique_id "Zabcdefg12345"]
Analyse : Cette ligne nous indique immédiatement que la règle avec l'ID 942100 a été déclenchée. Le message "SQL Injection Attack Detected" est explicite. L'ID unique est Zabcdefg12345.
Dans /var/log/apache2/modsec_audit.log, nous recherchons cet ID unique :
sudo grep "Zabcdefg12345" /var/log/apache2/modsec_audit.log
Résultat : La commande affichera l'entrée d'audit complète. En examinant la section B, nous verrons la ligne GET /?id=1' OR 1=1-- HTTP/1.1. En examinant la section H, nous verrons le message détaillé de la règle 942100 et comment le score d'anomalie a été incrémenté. Cela confirme précisément pourquoi la requête a été bloquée.

### 6.0 Section 6 : Réglage de base et gestion des faux positifs

Un WAF non réglé est une source de problèmes. Le réglage (tuning) est le processus d'adaptation du WAF au comportement normal de votre application afin de minimiser les faux positifs sans affaiblir la posture de sécurité.

#### 6.1 La stratégie de déploiement initial en mode DetectionOnly

La pratique la plus importante pour un déploiement en production réussi est de commencer avec SecRuleEngine DetectionOnly.2 Cette configuration permet à ModSecurity et au CRS d'analyser tout le trafic et de journaliser toutes les violations de règles qu'ils
auraient bloquées, mais sans jamais interférer avec le trafic réel. Cela crée une période de collecte de données sans risque, pendant laquelle l'administrateur peut observer le comportement normal de l'application à travers le prisme du WAF et identifier les règles qui se déclenchent sur du trafic légitime.

#### 6.2 Création d'un fichier d'exclusion de règles personnalisé pour des dérogations ciblées

Il ne faut jamais modifier directement les fichiers de règles du CRS (/etc/apache2/modsecurity-crs/rules/*.conf). Toute modification serait écrasée lors de la prochaine mise à jour du CRS. La bonne pratique consiste à créer des fichiers d'exclusion personnalisés. Le CRS est conçu pour charger automatiquement de tels fichiers.
Créez le fichier suivant :

`sudo touch /etc/apache2/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf`

Ce fichier sera chargé avant le reste des règles du CRS, ce qui en fait l'endroit idéal pour désactiver des règles spécifiques.

#### 6.3 Bonnes pratiques pour la désactivation de règles via SecRuleRemoveById

Le réglage d'un WAF doit être un processus systématique et documenté.
Déployer en mode DetectionOnly.
Surveiller les journaux (error.log et modsec_audit.log) pendant une période représentative (quelques jours à une semaine) pour collecter des données sur le trafic normal de l'application.
Identifier les faux positifs. Supposons qu'une fonctionnalité légitime de votre application envoie une requête qui déclenche la règle 942100 (injection SQL) parce qu'un champ de texte contient une apostrophe.
Créer une exclusion spécifique. Au lieu de désactiver la règle globalement, ce qui serait dangereux, la meilleure approche est de créer une exclusion ciblée. Pour commencer, on peut désactiver la règle par son ID. Ouvrez votre fichier d'exclusion personnalisé :

`sudo nano /etc/apache2/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf`

Ajouter une directive d'exclusion commentée.
Apache
# Désactivation de la règle 942100 pour l'application MyApp
# Raison : Faux positif sur le champ 'description' qui autorise les apostrophes.
# Date : 2025-10-27
# Analyste : J. Doe
`SecRuleRemoveById 942100`

Cette directive indique à ModSecurity d'ignorer complètement la règle avec l'ID 942100.3
Tester et valider. Après avoir ajouté l'exclusion, redémarrez Apache. Testez à nouveau la fonctionnalité légitime pour confirmer qu'elle n'est plus bloquée (ou signalée). Ensuite, ré-exécutez les simulations d'attaques (curl) pour vous assurer que l'exclusion n'a pas créé de nouvelle faille de sécurité.
Passer en mode blocage. Répétez ce processus pour tous les faux positifs identifiés. Une fois que les journaux sont "propres" (c'est-à-dire qu'aucun trafic légitime ne déclenche d'alertes), vous pouvez passer en toute confiance à SecRuleEngine On dans /etc/modsecurity/modsecurity.conf et redémarrer Apache.

## Partie III : Conclusion et étapes suivantes


### 7.0 Résumé des bonnes pratiques et voie vers le réglage avancé

La mise en place d'un pare-feu applicatif web efficace avec ModSecurity et l'OWASP Core Rule Set est un processus méthodique qui va bien au-delà de la simple installation d'un paquet logiciel. Les points clés à retenir sont :
La distinction fondamentale entre le moteur et les règles : ModSecurity est le moteur, mais l'OWASP CRS est l'intelligence. L'un sans l'autre est inefficace.
L'importance de la source : Utilisez toujours la dernière version stable de l'OWASP CRS provenant du dépôt GitHub officiel coreruleset.
La stratégie DetectionOnly d'abord : Ne jamais déployer un WAF en mode de blocage directement en production. Une phase d'observation est non négociable pour éviter les interruptions de service.
Le flux de travail de l'analyse des journaux : Utilisez le error.log comme signal et le modsec_audit.log comme contexte pour une analyse efficace des événements.
Le processus de réglage structuré : Utilisez des fichiers d'exclusion personnalisés pour gérer les faux positifs de manière propre et maintenable.
Ce guide a couvert les fondations essentielles pour un déploiement robuste. La sécurité est un processus continu, et la prochaine étape consiste à explorer des sujets plus avancés. Ceux-ci incluent l'écriture de règles personnalisées pour protéger des fonctionnalités spécifiques de votre application 11, la création d'exclusions plus granulaires qui ne désactivent une règle que pour une URL ou un paramètre spécifique (
SecRuleUpdateTargetById), et l'intégration des journaux de ModSecurity dans un système centralisé de gestion des informations et des événements de sécurité (SIEM) comme la suite Elastic (ELK) ou Splunk pour une corrélation et une alerte avancées. En maîtrisant ces bases, les administrateurs et les équipes de sécurité disposent d'une puissante couche de défense pour protéger leurs applications web contre un paysage de menaces en constante évolution.
Sources des citations
Apache2 : installer le ModSecurity (WAF) - RDR-IT, consulté le septembre 23, 2025, https://rdr-it.com/apache2-installer-le-modsecurity-waf/
Notes on installing ModSecurity and applying it to Apache (Ubuntu 24.04 LTS), consulté le septembre 23, 2025, https://beyondjapan.com/en/blog/2024/08/modsecurity/
How To Configure Apache Security on Ubuntu & Debian – TecAdmin, consulté le septembre 23, 2025, https://tecadmin.net/how-to-setup-apache-modsecurity-on-ubuntu/
How to Install the ModSecurity Apache Module - InMotion Hosting, consulté le septembre 23, 2025, https://www.inmotionhosting.com/support/server/apache/install-modsecurity-apache-module/
How to Install Apache Web Server on Ubuntu 24.04 - Vultr Docs, consulté le septembre 23, 2025, https://docs.vultr.com/how-to-install-apache-web-server-on-ubuntu-24-04
How to use Apache2 modules - Ubuntu Server documentation, consulté le septembre 23, 2025, https://documentation.ubuntu.com/server/how-to/web-services/use-apache2-modules/
owasp-modsecurity/ModSecurity: ModSecurity is an open source, cross platform web application firewall (WAF) engine for Apache, IIS and Nginx. It has a robust event-based programming language which provides protection from a range of attacks against web applications and allows for HTTP traffic monitoring, logging and real-time analysis - GitHub, consulté le septembre 23, 2025, https://github.com/owasp-modsecurity/ModSecurity
coreruleset/coreruleset: OWASP CRS (Official Repository) - GitHub, consulté le septembre 23, 2025, https://github.com/coreruleset/coreruleset
OWASP CRS, consulté le septembre 23, 2025, https://owasp.org/www-project-modsecurity-core-rule-set/
OWASP CRS, consulté le septembre 23, 2025, https://devguide.owasp.org/en/09-operations/04-crs/
How to Install and Configure ModSecurity on Apache for Ubuntu ..., consulté le septembre 23, 2025, https://medium.com/@redswitches/how-to-install-and-configure-modsecurity-on-apache-for-ubuntu-6d059400347c
Comprehensive Guide to ModSecurity: Logs, Configuration, and Important Limits, consulté le septembre 23, 2025, https://www.domainindia.com/login/knowledgebase/639/Comprehensive-Guide-to-ModSecurity-Logs-Configuration-and-Important-Limits.html
ModSecurity Handbook: Getting Started: Chapter 4. Logging - Feisty Duck, consulté le septembre 23, 2025, https://www.feistyduck.com/library/modsecurity-handbook-2ed-free/online/ch04-logging.html
Upgrading the provided OWASP Core Rule Set of ModSecurity - Nevis documentation, consulté le septembre 23, 2025, https://docs.nevis.net/configurationguide/use-cases/Application-Protection/Protecting-a-Web-Application/Upgrading-the-provided-OWASP-Core-Rule-Set-of-ModSecurity
OWASP ModSecurity Core Rule Set (CRS) Project (Official Repository) - GitHub, consulté le septembre 23, 2025, https://github.com/SpiderLabs/owasp-modsecurity-crs
CRS versions 4.8.0 and 3.3.7 released, consulté le septembre 23, 2025, https://coreruleset.org/20241029/crs-versions-4-8-0-and-3-3-7-released/
Security Overview · coreruleset/coreruleset - GitHub, consulté le septembre 23, 2025, https://github.com/coreruleset/coreruleset/security
Releases · coreruleset/coreruleset - GitHub, consulté le septembre 23, 2025, https://github.com/coreruleset/coreruleset/releases
OWASP CRS Project, consulté le septembre 23, 2025, https://coreruleset.org/
NGINX + ModSecurity v3 + OWASP CRS on Ubuntu 24.04 LTS – Step by Step – Part 2, consulté le septembre 23, 2025, https://www.softworx.at/en/nginx-modsecurity-v3-owasp-crs-on-ubuntu-24-04-lts-step-by-step-part-2/
CRS Installation :: CRS Documentation - OWASP CRS Project, consulté le septembre 23, 2025, https://coreruleset.org/docs/1-getting-started/1-1-crs-installation/
mod security - Test whether mod_security is actually working - Server Fault, consulté le septembre 23, 2025, https://serverfault.com/questions/642804/test-whether-mod-security-is-actually-working
ModSecurity: Logging and Debugging - F5, consulté le septembre 23, 2025, https://www.f5.com/company/blog/nginx/modsecurity-logging-and-debugging
ModSecurity Handbook: Getting Started: Chapter 4. Logging - Feisty Duck, consulté le septembre 23, 2025, https://www.feistyduck.com/library/modsecurity-handbook-free/online/ch04-logging.html
