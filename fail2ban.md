# **Fail2ban : Configuration et validation par Tests d'Intrusion**

## **Partie 1 : Installation et Configuration Fondamentale de Fail2ban**

### **1.1. Prérequis et Installation sur Debian/Ubuntu**

Avant d'installer tout logiciel de sécurité, il est impératif de s'assurer que le système d'exploitation sous-jacent est entièrement à jour. Cette étape préliminaire permet de corriger les vulnérabilités connues et d'assurer la compatibilité avec les paquets les plus récents. La procédure standard sur les systèmes basés sur Debian, comme Ubuntu, consiste à rafraîchir la liste des paquets et à appliquer les mises à jour disponibles.1

```Bash
sudo apt update && sudo apt upgrade \-y
```

Une fois le système mis à jour, l'installation de Fail2ban peut être effectuée via le gestionnaire de paquets apt. Il est recommandé d'installer explicitement iptables en même temps, car Fail2ban s'appuie sur le pare-feu du système pour appliquer les bannissements, et bien que les systèmes modernes migrent vers nftables, certaines dépendances ou scripts par défaut peuvent encore faire référence à iptables.1

```Bash
sudo apt install fail2ban iptables
```

Après l'installation, le service Fail2ban démarre automatiquement en arrière-plan.4 La première étape de vérification consiste à consulter son statut à l'aide de

systemctl pour confirmer qu'il est bien actif et en cours d'exécution.

```Bash
sudo systemctl status fail2ban
```

La sortie de cette commande devrait indiquer Active: active (running). Si le service n'est pas actif, il peut être démarré manuellement avec sudo systemctl start fail2ban.

Pour garantir que la protection offerte par Fail2ban persiste après un redémarrage du serveur, il est essentiel d'activer le service au démarrage du système.2

```Bash
sudo systemctl enable fail2ban
```

Cette commande crée les liens symboliques nécessaires pour que systemd lance le démon Fail2ban lors du processus de démarrage.

### **1.2. Architecture de Configuration : La Stratégie des Fichiers .local**

Comprendre l'architecture de configuration de Fail2ban est fondamental pour une administration efficace et pérenne. Les fichiers de configuration sont principalement situés dans le répertoire /etc/fail2ban/.1 Fail2ban utilise un système de surcharge hiérarchique pour lire ses paramètres, ce qui permet des mises à jour fluides sans écraser les personnalisations de l'administrateur. L'ordre de lecture est le suivant 9:

1. /etc/fail2ban/jail.conf  
2. /etc/fail2ban/jail.d/\*.conf (par ordre alphabétique)  
3. /etc/fail2ban/jail.local  
4. /etc/fail2ban/jail.d/\*.local (par ordre alphabétique)

Les paramètres lus dans les fichiers ultérieurs de cette liste écrasent ceux définis dans les fichiers précédents.9

La bonne pratique la plus importante dans la configuration de Fail2ban est de **ne jamais modifier directement les fichiers .conf**.1 Ces fichiers contiennent les configurations par défaut fournies par les mainteneurs du paquet. Lors d'une mise à jour de Fail2ban via le gestionnaire de paquets, ces fichiers

.conf peuvent être remplacés par de nouvelles versions, ce qui entraînerait la perte de toutes les modifications personnalisées.

Cette architecture de configuration n'est pas une simple commodité, mais un principe de conception fondamental visant à garantir la stabilité opérationnelle et la résilience de la posture de sécurité. Le système est délibérément conçu pour séparer les configurations par défaut des personnalisations de l'utilisateur. Cela permet aux mainteneurs de paquets de distribuer des mises à jour importantes — comme l'ajout de nouveaux filtres pour contrer de nouvelles menaces ou la correction de bogues dans les expressions régulières existantes — sans détruire les réglages spécifiques de l'administrateur. Ignorer cette pratique conduit inévitablement à un système fragile, soit parce que les personnalisations sont perdues à chaque mise à jour, rendant la sécurité inconstante, soit parce que les mises à jour doivent être fusionnées manuellement, un processus fastidieux et source d'erreurs. L'adoption de la stratégie des fichiers .local est donc une condition sine qua non pour une configuration de niveau production.

La méthode recommandée consiste à effectuer toutes les personnalisations dans des fichiers portant l'extension .local. Il existe deux approches principales :

* **Approche centralisée :** Créer un fichier /etc/fail2ban/jail.local qui contiendra toutes les surcharges des paramètres par défaut et les définitions des prisons personnalisées. Ce fichier n'a pas besoin de contenir l'intégralité de jail.conf, mais seulement les sections et les paramètres que l'on souhaite modifier.12  

* **Approche modulaire :** Créer des fichiers de configuration distincts pour chaque service dans le répertoire /etc/fail2ban/jail.d/. Par exemple, la configuration de la prison SSH pourrait être placée dans /etc/fail2ban/jail.d/sshd.local. Cette approche est souvent préférée pour sa clarté et sa facilité de gestion dans les environnements complexes.1

Pour ce laboratoire, nous utiliserons une approche mixte, en définissant les paramètres globaux dans jail.local et les configurations de prisons spécifiques dans des fichiers dédiés sous jail.d/.

### **1.3. Configuration Globale et Paramètres par Défaut dans jail.local**

Pour commencer, il est pratique de copier le fichier jail.conf vers jail.local afin de disposer d'un modèle complet de toutes les options disponibles.

```Bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Ensuite, éditez ce nouveau fichier avec un éditeur de texte (par exemple, nano).

```Bash
sudo nano /etc/fail2ban/jail.local
```

La section la plus importante à configurer est la section \`\`. Les paramètres définis ici s'appliquent à toutes les prisons, sauf s'ils sont spécifiquement surchargés dans la définition d'une prison individuelle.

* ignoreip : Ce paramètre est essentiel pour éviter les auto-bannissements. Il faut y ajouter les adresses IP des machines d'administration, les adresses de bouclage local et éventuellement les plages d'adresses du réseau local. Les adresses sont séparées par des espaces.3  
  

  ``` 
  ignoreip \= 127.0.0.1/8 ::1 VOTRE\_IP\_FIXE\_1 VOTRE\_IP\_FIXE\_2
  ```

* bantime : Définit la durée, en secondes, pendant laquelle une adresse IP sera bannie. La valeur par défaut de 600 secondes (10 minutes) est souvent jugée trop courte pour être dissuasive contre les attaques automatisées. Une valeur plus longue, comme 1h (une heure) ou 1d (un jour), est généralement recommandée.1  
* findtime : Spécifie la fenêtre de temps, en secondes, pendant laquelle Fail2ban va compter les tentatives infructueuses. Par exemple, si findtime \= 600, Fail2ban surveillera les tentatives survenues au cours des 10 dernières minutes.1  
* maxretry : C'est le nombre de tentatives échouées autorisées depuis une même adresse IP pendant la période findtime avant que le bannissement ne soit déclenché. Une valeur de 3 ou 5 est un bon compromis entre sécurité et tolérance aux erreurs humaines.1

Pour une surveillance proactive, il est également possible de configurer des notifications par email. Cela nécessite un agent de transfert de courrier (MTA) comme sendmail ou postfix installé sur le serveur. Les paramètres destemail, sender et action doivent être configurés dans la section \`\`.4

* destemail : L'adresse email du destinataire des alertes.  
* sender : L'adresse email de l'expéditeur.  
* action : L'action à exécuter. Pour recevoir un email avec un rapport whois et les lignes de log pertinentes, la valeur %(action\_mwl)s est utilisée.

Le tableau suivant résume les paramètres globaux essentiels et fournit des recommandations justifiées.

| Paramètre | Description | Valeur Recommandée | Justification |
| :---- | :---- | :---- | :---- |
| ignoreip | Adresses IP à ne jamais bannir. | 127.0.0.1/8 ::1 \<IP\_ADMIN\> | Prévention critique des auto-bannissements. |
| bantime | Durée du bannissement (secondes, m, h, d). | 1h | Un bannissement court (défaut de 10m) est peu dissuasif pour les attaques automatisées. 1 heure offre un bon compromis. |
| findtime | Fenêtre de temps pour la détection (secondes, m, h). | 10m | Une fenêtre de 10 minutes est suffisante pour détecter les attaques par force brute sans pénaliser un utilisateur légitime faisant quelques erreurs. |
| maxretry | Nombre d'échecs avant bannissement. | 3 ou 5 | Un maxretry bas (3) est agressif et efficace contre les bots. Un maxretry de 5 est plus tolérant pour les utilisateurs humains. |
| backend | Moteur de surveillance des logs. | systemd (pour Debian 10+) | Assure la compatibilité avec le journal systemd, source de logs primaire sur les systèmes modernes. |
| banaction | Action de bannissement du pare-feu. | nftables-multiport ou iptables-multiport | Doit correspondre au pare-feu actif sur le système. |

### **1.4. Intégration avec le Pare-feu : iptables vs. nftables**

Fail2ban n'est pas un pare-feu en soi ; c'est un automate qui pilote le pare-feu du système d'exploitation pour bloquer et débloquer les adresses IP.1 Par conséquent, son efficacité dépend entièrement de sa bonne intégration avec le pare-feu en place. Les deux principaux systèmes de pare-feu sous Linux sont

iptables, le standard historique, et nftables, son successeur plus moderne et performant, qui est devenu la norme sur les distributions récentes comme Debian 11 et ses dérivés.

Le choix du banaction (action de bannissement) et du backend (source des logs) est une dépendance critique qui reflète l'évolution de l'écosystème Linux. Une mauvaise configuration de ces paramètres est une cause fréquente d'échec silencieux, où Fail2ban semble fonctionner mais n'offre aucune protection réelle. Par exemple, sur un système Debian 12, les logs d'authentification SSH sont principalement gérés par le journal de systemd et le pare-feu est nftables. Si un administrateur laisse la configuration par défaut de Fail2ban, qui pourrait supposer une lecture du fichier /var/log/auth.log (backend \= auto) et une interaction avec iptables (banaction \= iptables-multiport), le service s'exécutera sans erreur apparente. Cependant, il ne lira jamais les tentatives de connexion échouées depuis le journal et ses tentatives d'ajout de règles iptables n'auront aucun effet sur le pare-feu nftables actif. Le système reste donc vulnérable malgré la présence d'un service de sécurité "actif".

Il est donc impératif de configurer explicitement ces paramètres dans /etc/fail2ban/jail.local pour qu'ils correspondent à l'environnement du serveur.

* Pour les systèmes utilisant iptables (généralement les distributions plus anciennes) :  
  ```
  banaction \= iptables-multiport
  ```

* Pour les systèmes modernes utilisant nftables :  
  ```
  banaction \= nftables-multiport
  ```



De même, le paramètre backend doit être ajusté pour indiquer à Fail2ban où chercher les logs. Sur les systèmes basés sur systemd, il est fortement recommandé de le définir explicitement.1


```
backend \= systemd
```

Après avoir effectué ces configurations fondamentales, il est nécessaire de redémarrer le service Fail2ban pour qu'elles prennent effet.

```Bash
sudo systemctl restart fail2ban
```

## **Partie 2 : Sécurisation des Services Critiques via les Prisons (Jails)**

Cette section constitue le cœur de la configuration de Fail2ban, où des politiques de sécurité ciblées sont mises en œuvre pour les services les plus exposés : l'accès à distance via SSH et les serveurs web. Chaque "prison" (jail) sera configurée avec précision, en définissant un filtre (quoi chercher), un chemin de log (où chercher) et une action (que faire). L'objectif est de bloquer les menaces spécifiques à chaque service tout en minimisant le risque de bloquer des utilisateurs légitimes (faux positifs).

### **2.1. Blindage du Service SSH (\[sshd\])**

Le service SSH est la cible la plus fréquente des attaques par force brute sur les serveurs Linux. La prison \[sshd\] est donc la plus importante à configurer et est souvent la seule activée par défaut sur de nombreuses distributions.7

Pour une gestion propre et modulaire, la configuration de cette prison sera placée dans un fichier dédié : /etc/fail2ban/jail.d/sshd.local.1

```Bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

Le contenu suivant constitue une configuration robuste et recommandée 1 :

```
\[sshd\]  
enabled \= true  
port \= ssh  
maxretry \= 3  
findtime \= 5m  
bantime \= 1h  
logpath \= %(sshd\_log)s  
backend \= %(sshd\_backend)s
```

**Analyse de la configuration :**

* enabled \= true : Active la prison. Sans cette ligne, la configuration est ignorée.  
* port \= ssh : Indique le port à surveiller. La valeur ssh est un alias pour le port 22\. Si le port SSH a été modifié pour des raisons de sécurité (une pratique recommandée), il est impératif de mettre à jour cette valeur. Par exemple : port \= 2222\.3  
* maxretry \= 3 : Un seuil bas et agressif, adapté pour bloquer rapidement les bots automatisés.  
* findtime \= 5m : Une fenêtre de 5 minutes. Si 3 tentatives échouées se produisent dans cet intervalle, l'IP est bannie.  
* bantime \= 1h : L'IP est bannie pour une heure, ce qui est suffisamment long pour décourager la plupart des attaques non ciblées.  
* logpath \= %(sshd\_log)s et backend \= %(sshd\_backend)s : Ces variables utilisent les chemins et backends définis par Fail2ban pour le service sshd, assurant la compatibilité entre différentes distributions. Sur les systèmes récents, %(sshd\_backend)s se résout généralement en systemd, ce qui est crucial pour une détection fiable des logs.1

### **2.2. Protection Avancée du Serveur Web (Nginx/Apache)**

La protection d'un serveur web avec Fail2ban est fondamentalement plus complexe que celle de SSH. Alors que SSH présente un vecteur d'attaque principal (l'échec d'authentification), un serveur web expose une surface d'attaque beaucoup plus large. Les attaquants peuvent effectuer des scans de reconnaissance (générant des erreurs 404), des attaques par force brute sur des formulaires de connexion (comme /wp-login.php), ou tenter d'exploiter des vulnérabilités applicatives.

Une stratégie de défense efficace ne peut donc pas reposer sur une unique prison. Elle nécessite une approche multi-facettes, avec plusieurs prisons distinctes, chacune ciblant un comportement malveillant spécifique identifiable dans les journaux du serveur web (access.log et error.log).24

#### **2.2.1. Filtre et Prison contre les Scans de Vulnérabilités (Erreurs 404\)**

Les scans automatisés qui recherchent des fichiers de configuration, des back-ups ou des vulnérabilités connues génèrent un grand nombre de requêtes pour des ressources inexistantes, ce qui se traduit par des erreurs 404 (Not Found) ou 403 (Forbidden) dans les logs. Bien qu'une seule erreur 404 soit anodine, une succession rapide d'erreurs 404 provenant de la même IP est un indicateur fort d'une tentative de reconnaissance.

Pour contrer cela, une prison personnalisée est nécessaire. La première étape est de créer un filtre.

```Bash
sudo nano /etc/fail2ban/filter.d/nginx-4xx.conf
```

Ce filtre utilisera une expression régulière (failregex) pour identifier les lignes de log contenant les codes d'erreur 404 et 403\.25

```
failregex \= ^\<HOST\>.\*"(GET|POST).\*" (404|403).\*$
```

Ensuite, la prison correspondante est définie dans /etc/fail2ban/jail.d/nginx.local.24

```
\[nginx-4xx\]  
enabled  \= true  
port     \= http,https  
filter   \= nginx-4xx  
logpath  \= /var/log/nginx/access.log  
maxretry \= 10  
findtime \= 1m  
bantime  \= 1h
```


Cette configuration bannira pour une heure toute IP générant plus de 10 erreurs 4xx en moins d'une minute. Il est parfois utile d'ajouter un ignoreregex pour éviter de bannir des IP pour des fichiers manquants courants qui ne sont pas malveillants, comme des images ou des fichiers CSS.27

#### **2.2.2. Blocage des Attaques par Force Brute sur l'Authentification Web**

Pour les répertoires protégés par une authentification HTTP Basic (souvent configurée avec .htpasswd), Nginx enregistre les tentatives de connexion échouées dans son journal d'erreurs (error.log). Fail2ban fournit une prison préconfigurée, \[nginx-http-auth\], pour ce scénario précis.15

La configuration est ajoutée à /etc/fail2ban/jail.d/nginx.local :

```
\[nginx-http-auth\]  
enabled \= true  
port    \= http,https  
logpath \= %(nginx\_error\_log)s  
maxretry \= 3  
bantime  \= 1h
```

Cette prison utilise le filtre nginx-http-auth, qui recherche des messages tels que "no user/password was provided for basic authentication" ou "user... password mismatch" dans le fichier error.log de Nginx.15

#### **2.2.3. Cas d'Usage Spécifique : Sécurisation de WordPress (wp-login.php)**

La protection d'applications web complexes comme WordPress illustre une limitation fondamentale de Fail2ban et un principe clé de l'architecture de sécurité : la collaboration entre les outils. Fail2ban est un analyseur de logs ; il n'a aucune connaissance de la logique interne d'une application. Une tentative de connexion échouée sur WordPress ne génère pas, par défaut, une erreur unique et facilement identifiable dans les logs standards de Nginx. Le serveur retourne souvent un code de statut 200 OK avec un message d'erreur affiché dans le code HTML de la page, un format que Fail2ban ne peut pas analyser.

Pour rendre ces échecs visibles, un intermédiaire est nécessaire. Le plugin wp-fail2ban remplit précisément ce rôle.30 Il s'intègre au processus d'authentification de WordPress et, lors d'un échec, écrit un message clair et standardisé (par exemple,

Authentication failure for user from \<HOST\>) dans un journal système comme syslog ou /var/log/auth.log. C'est cette entrée de log, spécifiquement conçue pour être lue par une machine, que Fail2ban peut ensuite détecter et traiter. Ce cas d'usage démontre un modèle d'architecture de sécurité critique : les applications doivent être configurées pour produire des journaux d'événements de sécurité structurés afin que les outils de surveillance externes puissent être efficaces.

Après avoir installé et activé le plugin wp-fail2ban depuis l'interface d'administration de WordPress, il faut configurer une prison pour surveiller les logs qu'il génère.

D'abord, le filtre dans /etc/fail2ban/filter.d/wordpress.conf 31 :

```
failregex \= ^%(\_\_prefix\_line)sAuthentication failure for.\* from \<HOST\>$
```

Ensuite, la prison dans /etc/fail2ban/jail.d/wordpress.local 32 :

```
\[wordpress\]  
enabled  \= true  
port     \= http,https  
filter   \= wordpress  
logpath  \= /var/log/auth.log  
maxretry \= 5  
bantime  \= 24h
```

Cette configuration bannira pour 24 heures toute IP qui échoue à se connecter à WordPress plus de 5 fois. Le logpath pointe vers /var/log/auth.log, car c'est la destination par défaut des logs du plugin wp-fail2ban.

Après avoir ajouté ou modifié des prisons, un redémarrage de Fail2ban est nécessaire.

```Bash
sudo systemctl restart fail2ban
```

## **Partie 3 : Maintenance Opérationnelle et Commandes Essentielles**

Un outil de sécurité n'est efficace que s'il est correctement géré et surveillé. Cette section fournit un guide de référence pratique pour l'administration quotidienne de Fail2ban, permettant de diagnostiquer les problèmes, de répondre aux incidents et de maintenir le système en état de fonctionnement optimal.

### **3.1. Maîtrise de l'Utilitaire fail2ban-client**

L'utilitaire en ligne de commande fail2ban-client est l'interface principale pour interagir avec le démon Fail2ban en cours d'exécution.36 Il permet de consulter l'état, de modifier la configuration à la volée et d'effectuer des actions manuelles de bannissement ou de dé-bannissement.

Les commandes de base pour la gestion du service incluent :

* ping : Vérifie que le serveur Fail2ban est en cours d'exécution et répond. La réponse attendue est pong.36  
* reload : Recharge les fichiers de configuration. Cela permet d'appliquer des modifications sans interrompre la surveillance. La syntaxe est fail2ban-client reload ou fail2ban-client reload \<JAIL\> pour une prison spécifique.17  
* restart : Redémarre complètement le service Fail2ban.

Il existe une nuance opérationnelle critique entre reload et restart. Plusieurs retours d'expérience indiquent que reload (ou systemctl reload fail2ban) n'applique pas toujours correctement certains changements de configuration majeurs, comme la modification du banaction (passage de iptables à nftables) ou du backend.38 Dans ces cas, seul un restart complet garantit que la nouvelle configuration est entièrement chargée et appliquée. Cette subtilité est un piège potentiel pour les administrateurs. Cela implique que pour tout changement critique de sécurité, un redémarrage complet du service devrait être la procédure par défaut pour garantir son application, suivi d'une vérification rigoureuse, plutôt que de se fier à un rechargement potentiellement incomplet.

### **3.2. Scénarios de Maintenance Courants**

Voici les commandes les plus utiles pour les tâches d'administration quotidiennes.

* Vérifier le statut global et les prisons actives :  
  La commande status sans argument liste toutes les prisons actuellement actives.17  
  
```Bash  
  sudo fail2ban-client status
 ``` 
 ```
  Exemple de sortie :  
  Status

|- Number of jail: 3  
\`- Jail list: sshd, nginx-4xx, wordpress  
\`\`\`
```

* Inspecter une prison spécifique :  
  Pour obtenir des informations détaillées sur une prison, y compris le nombre d'échecs actuels, le nombre total de bannissements et la liste des adresses IP actuellement bannies, on utilise status suivi du nom de la prison.1  
  
   ```Bash  
  sudo fail2ban-client status sshd
   ```

```
  Exemple de sortie :  
  Status for the jail: sshd
|- Filter  
| |- Currently failed: 0  
| |- Total failed: 42  
| \- Journal matches: \_SYSTEMD\_UNIT=sshd.service \+ \_COMM=sshd \- Actions  
|- Currently banned: 1  
|- Total banned: 5  
\`- Banned IP list: 203.0.113.10  
\`\`\`
```

* Bannir manuellement une adresse IP :  
  Si une adresse IP est identifiée comme malveillante par d'autres moyens (par exemple, via les logs d'une application), elle peut être bannie manuellement sans attendre une détection automatique. La commande set \<JAIL\> banip est utilisée à cet effet.5  
  ```Bash  
  sudo fail2ban-client set sshd banip 203.0.113.25
  ```

* Débannir une adresse IP :  
  C'est une opération courante lorsqu'un utilisateur légitime est banni par erreur. La commande set \<JAIL\> unbanip retire immédiatement le blocage.1  
  ```Bash  
  sudo fail2ban-client set sshd unbanip 203.0.113.10
  ```

* Lister toutes les IP bannies (via les logs) :  
  Bien que fail2ban-client status soit la méthode canonique pour voir les bannissements actuels, l'analyse des logs de Fail2ban permet d'obtenir un historique des actions de bannissement. Des commandes shell peuvent être utilisées pour extraire et compter ces occurrences.43  
 
  ```Bash  
  \# Affiche toutes les lignes de bannissement du log actuel  
  sudo grep 'Ban' /var/log/fail2ban.log
  
  \# Compte le nombre de fois que chaque IP a été bannie et les trie  
  sudo zgrep 'Ban' /var/log/fail2ban.log\* | awk '{print $NF}' | sort | uniq \-c | sort \-nr
  ```

Le tableau suivant sert de mémento pour les commandes fail2ban-client les plus critiques. En organisant ces commandes par fonction et en fournissant une syntaxe claire et des cas d'usage, il réduit la charge cognitive de l'administrateur, améliorant l'efficacité et la précision lors de la maintenance de routine et de la réponse aux incidents.

| Commande | Syntaxe | Objectif | Cas d'Usage Typique |
| :---- | :---- | :---- | :---- |
| status | fail2ban-client status | Voir la liste des prisons actives. | Vérification rapide que les bonnes prisons sont en cours d'exécution. |
| status \<JAIL\> | fail2ban-client status sshd | Obtenir les détails d'une prison (IPs bannies, etc.). | Diagnostiquer un problème de blocage sur un service spécifique. |
| reload | fail2ban-client reload | Recharger les fichiers de configuration sans redémarrer le service. | Appliquer des modifications mineures (ex: changer un bantime). **Attention : un restart est plus fiable pour les changements majeurs.** |
| unbanip | fail2ban-client set sshd unbanip 1.2.3.4 | Retirer une IP de la liste des bannis pour une prison. | Un utilisateur légitime a été bloqué par erreur. |
| banip | fail2ban-client set sshd banip 1.2.3.4 | Bannir manuellement une IP pour une prison. | Une IP a été identifiée comme malveillante par une autre source et doit être bloquée immédiatement. |
| ping | fail2ban-client ping | Vérifier que le démon fail2ban est en cours d'exécution et répond. | Première étape de dépannage pour s'assurer que le service est fonctionnel. |

## **Partie 4 : Laboratoire de Test d'Intrusion (Validation des Défenses)**

La configuration de mesures de sécurité sans une validation empirique rigoureuse n'est qu'une hypothèse. Cette section cruciale transforme la théorie en pratique en simulant des attaques réelles depuis une machine dédiée (Kali Linux) pour vérifier que les défenses Fail2ban sont non seulement actives, mais aussi efficaces. Ce processus de validation est indispensable pour détecter les "échecs silencieux" et garantir que la chaîne de détection et de réponse fonctionne comme prévu.

### **4.1. Mise en Place de l'Environnement de Test**

L'environnement de test est simple mais efficace. Il se compose de deux machines virtuelles ou physiques sur le même réseau 4 :

1. **Serveur Cible :** Une machine Debian ou Ubuntu sur laquelle Fail2ban a été installé et configuré conformément aux sections précédentes. Noter son adresse IP.  
2. **Machine Attaquante :** Une machine Kali Linux, qui dispose d'une suite complète d'outils de test d'intrusion préinstallés, comme hydra et nikto. Noter son adresse IP.

### **4.2. Scénario 1 : Attaque par Force Brute sur SSH avec Hydra**

Ce scénario teste l'efficacité de la prison \[sshd\]. L'objectif est de simuler un attaquant tentant de deviner le mot de passe d'un utilisateur SSH.

#### **Phase d'Attaque (Kali Linux)**

L'outil hydra est un craqueur de mots de passe en ligne très rapide, capable de lancer des attaques par dictionnaire contre de nombreux services, y compris SSH.50 La commande suivante lance une attaque sur le serveur cible, en essayant une liste de mots de passe pour l'utilisateur root.

```Bash
\# Sur la machine Kali Linux  
hydra \-l root \-P /usr/share/wordlists/rockyou.txt ssh://\<IP\_CIBLE\>

* \-l root : Spécifie le nom d'utilisateur à attaquer.  
* \-P /usr/share/wordlists/rockyou.txt : Spécifie le chemin vers une liste de mots de passe.  
* ssh://\<IP\_CIBLE\> : Définit le protocole et l'adresse IP de la cible.
```

#### **Phase de Vérification (Serveur Cible)**

La vérification se déroule en plusieurs étapes pour confirmer que chaque maillon de la chaîne de défense a fonctionné.

1. **Observation en temps réel :** L'exécution de hydra sur la machine Kali devrait s'interrompre brusquement après quelques tentatives (correspondant au maxretry de la prison sshd, soit 3 dans notre configuration). L'outil affichera probablement des erreurs de connexion ou de "timeout".  
2. **Analyse des logs Fail2ban :** Sur le serveur cible, surveiller le log de Fail2ban en temps réel.  

```Bash  
   sudo tail \-f /var/log/fail2ban.log
```

   Une ligne de notification de bannissement devrait apparaître, indiquant que l'adresse IP de la machine Kali a été bannie par la prison sshd.5

```
... fail2ban.actions\[...\]: NOTICE \[sshd\] Ban \<IP\_KALI\>  
\`\`\`
```

3. **Vérification du statut de la prison :** Utiliser fail2ban-client pour interroger directement l'état de la prison sshd.  
   ```Bash  
   sudo fail2ban-client status sshd
   ```
   La sortie doit explicitement lister l'adresse IP de la machine Kali dans la Banned IP list.1 

4. **Inspection du pare-feu :** C'est la preuve ultime et irréfutable du blocage. La commande dépend du pare-feu utilisé. 

   * **Avec iptables :** Lister les règles de la chaîne créée par Fail2ban pour la prison sshd. Le nom de la chaîne est généralement f2b-sshd.1  
     ```Bash  
     sudo iptables \-L f2b-sshd \-n
     ```

     La sortie doit montrer une règle REJECT ou DROP pour l'adresse IP source \<IP\_KALI\>. 

   * **Avec nftables :** Lister l'ensemble des règles. La structure exacte peut varier, mais Fail2ban crée généralement une table (par exemple, fail2ban) et des chaînes dédiées.17  
     ```Bash  
     sudo nft list ruleset

     Il faut rechercher dans la sortie une règle qui bloque le trafic provenant de \<IP\_KALI\> sur le port SSH.
     ```

### **4.3. Scénario 2 : Scan de Vulnérabilités Web avec Nikto**

Ce scénario valide l'efficacité de la prison \[nginx-4xx\] en simulant un scan de reconnaissance automatisé sur le serveur web Nginx.

#### **Phase d'Attaque (Kali Linux)**

Nikto est un scanner de vulnérabilités web qui effectue des milliers de tests en envoyant des requêtes pour des fichiers et des répertoires connus pour être dangereux ou informatifs. Ce comportement génère inévitablement un grand nombre d'erreurs 404\.54

```Bash
\# Sur la machine Kali Linux  
nikto \-h http://\<IP\_CIBLE\>
```

#### **Phase de Vérification (Serveur Cible)**

Le processus de vérification est similaire à celui du scénario SSH.

1. **Observation en temps réel :** Le scan nikto devrait s'arrêter prématurément en raison d'une perte de connectivité avec le serveur cible.  
2. **Analyse des logs Nginx :** Surveiller le journal d'accès de Nginx.  
   ```Bash  
   sudo tail \-f /var/log/nginx/access.log
   ```

   On devrait observer une série de requêtes pour divers chemins, toutes résultant en un code de statut 404, provenant de l'adresse IP de la machine Kali.  

3. **Analyse des logs Fail2ban :** Le log de Fail2ban doit afficher une notification de bannissement pour la prison nginx-4xx.

```
... fail2ban.actions\[...\]: NOTICE \[nginx-4xx\] Ban \<IP\_KALI\>  
\`\`\`
```

4. **Vérification du statut de la prison :**  
   ```Bash  
   sudo fail2ban-client status nginx-4xx
   ```

   L'adresse IP de Kali doit figurer dans la liste des IP bannies.  

5. **Inspection du pare-feu :** Vérifier la présence d'une règle iptables ou nftables bloquant \<IP\_KALI\> sur les ports HTTP/HTTPS (80 et 443).

### **4.4. Analyse et Interprétation des Résultats**

La réussite de ces tests valide l'ensemble de la chaîne de défense mise en place. La boucle de validation est la suivante :

1. **Attaque :** Une activité malveillante est simulée.  
2. **Détection :** Le service ciblé (SSHD, Nginx) enregistre l'activité anormale dans ses journaux.  
3. **Action :** Le démon Fail2ban, en surveillant les journaux, détecte le modèle d'attaque correspondant à un filtre.  
4. **Application :** Fail2ban invoque l'action de bannissement configurée (banaction).  
5. **Blocage :** L'action ajoute une règle au pare-feu du système (iptables ou nftables).  
6. **Validation :** L'attaquant est incapable de poursuivre son attaque car ses paquets sont désormais bloqués au niveau du réseau.

Cette validation empirique est ce qui transforme une configuration de sécurité théorique en une défense éprouvée et fiable. Elle fournit la preuve tangible que le système est capable de répondre activement aux menaces automatisées.

## **Conclusion et Recommandations Stratégiques**

Ce laboratoire pratique a détaillé le processus complet de déploiement, de configuration et de validation de Fail2ban, un outil essentiel pour la protection proactive des serveurs Linux. En suivant les étapes décrites, un administrateur peut acquérir les compétences nécessaires pour installer, configurer de manière robuste, maintenir et, surtout, vérifier l'efficacité d'un système de prévention d'intrusion basé sur l'analyse des journaux.

L'analyse a démontré que l'efficacité de Fail2ban ne réside pas seulement dans son installation, mais dans une configuration méticuleuse qui respecte son architecture de fichiers .local, qui est adaptée aux technologies sous-jacentes du système d'exploitation (comme systemd et nftables), et qui est validée par des tests rigoureux. Les scénarios d'attaque simulés avec hydra et nikto ont prouvé que des prisons bien configurées pour SSH et les serveurs web peuvent détecter et bloquer efficacement les menaces automatisées courantes.

Il est crucial de comprendre que Fail2ban est un composant d'une stratégie de **défense en profondeur**, et non une solution miracle. Son rôle est de servir de première ligne de défense active contre les attaques automatisées à faible et moyenne sophistication, réduisant ainsi la charge sur les services et le bruit dans les journaux. Cependant, il ne remplace en aucun cas les mesures de sécurité fondamentales telles que l'utilisation de mots de passe forts, l'authentification par clé publique pour SSH, la configuration d'un pare-feu restrictif, et la mise à jour régulière des logiciels pour corriger les vulnérabilités.1

Pour aller plus loin et renforcer davantage la posture de sécurité, les recommandations stratégiques suivantes peuvent être envisagées :

* **Mise en place d'un bannissement progressif :** Configurer la prison spéciale \[recidive\]. Cette prison analyse les logs de Fail2ban lui-même et peut imposer des bannissements beaucoup plus longs (voire permanents) aux adresses IP qui sont bannies de manière répétée sur différentes prisons, ciblant ainsi les attaquants les plus persistants.20  
* **Intégration avec des services de réputation d'IP :** Développer des actions personnalisées qui, en plus de bannir localement une IP, la signalent à des listes de réputation partagées comme AbuseIPDB, contribuant ainsi à l'écosystème de la cybersécurité.  
* **Centralisation des logs :** Dans un environnement comprenant plusieurs serveurs, la centralisation des logs (via des outils comme rsyslog, Elastic Stack ou Graylog) permet de faire fonctionner une instance de Fail2ban qui peut corréler les attaques sur l'ensemble de l'infrastructure et prendre des décisions de bannissement plus globales.

En conclusion, Fail2ban, lorsqu'il est correctement mis en œuvre et validé, est un outil extrêmement puissant et rentable pour améliorer de manière significative la résilience d'un serveur face au paysage de menaces constant d'Internet.

#### **Sources des citations**

1. Configuring Fail2ban in Debian/Ubuntu \- Knowledge Base \- EuroHoster, consulté le septembre 26, 2025, [https://eurohoster.org/en/knowledgebase/1287/Configuring+Fail2ban+in+Debian+Ubuntu.html](https://eurohoster.org/en/knowledgebase/1287/Configuring+Fail2ban+in+Debian+Ubuntu.html)  
2. How to Use Fail2Ban for SSH Brute-force Protection | Linode Docs, consulté le septembre 26, 2025, [https://www.linode.com/docs/guides/how-to-use-fail2ban-for-ssh-brute-force-protection/](https://www.linode.com/docs/guides/how-to-use-fail2ban-for-ssh-brute-force-protection/)  
3. How to Install and Use fail2ban in Ubuntu and Debian \- Atlantic.Net, consulté le septembre 26, 2025, [https://www.atlantic.net/vps-hosting/how-to-install-fail2ban-ubuntu-debian/](https://www.atlantic.net/vps-hosting/how-to-install-fail2ban-ubuntu-debian/)  
4. How To Protect SSH with Fail2Ban on Ubuntu 20.04 | DigitalOcean, consulté le septembre 26, 2025, [https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04)  
5. Installing and Configuring Fail2ban to secure SSH | by Binaya Sharma \- Medium, consulté le septembre 26, 2025, [https://medium.com/@bnay14/installing-and-configuring-fail2ban-to-secure-ssh-1e4e56324b19](https://medium.com/@bnay14/installing-and-configuring-fail2ban-to-secure-ssh-1e4e56324b19)  
6. How to Install and configure Fail2ban \- SecOps® Solution, consulté le septembre 26, 2025, [https://www.secopsolution.com/blog/install-and-configure-fail2ban](https://www.secopsolution.com/blog/install-and-configure-fail2ban)  
7. Fail2ban Server Security Guide: Installation & Configuration \- Plesk, consulté le septembre 26, 2025, [https://www.plesk.com/blog/various/using-fail2ban-to-secure-your-server/](https://www.plesk.com/blog/various/using-fail2ban-to-secure-your-server/)  
8. Fail2ban needs restart before running on ubuntu 22.04 \- Ask Ubuntu, consulté le septembre 26, 2025, [https://askubuntu.com/questions/1464600/fail2ban-needs-restart-before-running-on-ubuntu-22-04](https://askubuntu.com/questions/1464600/fail2ban-needs-restart-before-running-on-ubuntu-22-04)  
9. fail2ban | difference between \[sshd\] in jail.local, vs sshd.local in jail.d? \- Server Fault, consulté le septembre 26, 2025, [https://serverfault.com/questions/1151920/fail2ban-difference-between-sshd-in-jail-local-vs-sshd-local-in-jail-d](https://serverfault.com/questions/1151920/fail2ban-difference-between-sshd-in-jail-local-vs-sshd-local-in-jail-d)  
10. jail.conf \- configuration for the fail2ban server \- Ubuntu Manpage, consulté le septembre 26, 2025, [https://manpages.ubuntu.com/manpages/xenial/man1/jail.conf.10.html](https://manpages.ubuntu.com/manpages/xenial/man1/jail.conf.10.html)  
11. jail.conf: configuration for the fail2ban server \- ManKier, consulté le septembre 26, 2025, [https://www.mankier.com/5/jail.conf](https://www.mankier.com/5/jail.conf)  
12. Fail2ban jail.local vs jail.conf \- iptables \- Server Fault, consulté le septembre 26, 2025, [https://serverfault.com/questions/639923/fail2ban-jail-local-vs-jail-conf](https://serverfault.com/questions/639923/fail2ban-jail-local-vs-jail-conf)  
13. Configure Fail2Ban for services → Great docs » Webdock.io, consulté le septembre 26, 2025, [https://webdock.io/en/docs/how-guides/security-guides/how-configure-fail2ban-common-services](https://webdock.io/en/docs/how-guides/security-guides/how-configure-fail2ban-common-services)  
14. How To Install Fail2ban On Debian \- UpCloud, consulté le septembre 26, 2025, [https://upcloud.com/resources/tutorials/install-fail2ban-debian/](https://upcloud.com/resources/tutorials/install-fail2ban-debian/)  
15. Securing NGINX and SSH Servers on Ubuntu Using Fail2Ban \- zenarmor.com, consulté le septembre 26, 2025, [https://www.zenarmor.com/docs/linux-tutorials/how-to-secure-nginx-and-ssh-server-using-fail2ban](https://www.zenarmor.com/docs/linux-tutorials/how-to-secure-nginx-and-ssh-server-using-fail2ban)  
16. Bannir des IP avec fail2ban \- Wiki ubuntu-fr, consulté le septembre 26, 2025, [https://doc.ubuntu-fr.org/fail2ban](https://doc.ubuntu-fr.org/fail2ban)  
17. Fail2ban \- ArchWiki, consulté le septembre 26, 2025, [https://wiki.archlinux.org/title/Fail2ban](https://wiki.archlinux.org/title/Fail2ban)  
18. fail2ban : Configuration pour fonctionner avec systemd et nftables \- echo42.fr, consulté le septembre 26, 2025, [https://echo42.fr/posts/fail2ban-configuration-pour-fonctionner-avec-systemd-et-nftables/](https://echo42.fr/posts/fail2ban-configuration-pour-fonctionner-avec-systemd-et-nftables/)  
19. fail2ban nftables Configuration \- gbe0.com Wiki, consulté le septembre 26, 2025, [https://wiki.gbe0.com/linux/firewalling-and-filtering/nftables/fail2ban](https://wiki.gbe0.com/linux/firewalling-and-filtering/nftables/fail2ban)  
20. Fail2ban with nftables and IPv6 \- Server Fault, consulté le septembre 26, 2025, [https://serverfault.com/questions/873068/fail2ban-with-nftables-and-ipv6](https://serverfault.com/questions/873068/fail2ban-with-nftables-and-ipv6)  
21. Debian: fail2ban \+ nftables \- Cyberfront Tech Blog, consulté le septembre 26, 2025, [https://blog.cyberfront.org/index.php/2021/10/27/debian-fail2ban/](https://blog.cyberfront.org/index.php/2021/10/27/debian-fail2ban/)  
22. Debian 12 bookworm and nftables won't block \#3575 \- GitHub, consulté le septembre 26, 2025, [https://github.com/fail2ban/fail2ban/discussions/3575](https://github.com/fail2ban/fail2ban/discussions/3575)  
23. Comment installer et configurer Fail2ban sur Linux \- Tutoriel & Documentation, consulté le septembre 26, 2025, [https://www.webhi.com/how-to/fr/comment-installer-et-configurer-fail2ban-sur-linux/](https://www.webhi.com/how-to/fr/comment-installer-et-configurer-fail2ban-sur-linux/)  
24. How To Protect an Nginx Server with Fail2Ban on Ubuntu 22.04 ..., consulté le septembre 26, 2025, [https://shape.host/resources/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-22-04](https://shape.host/resources/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-22-04)  
25. fail2ban regular to find 403 request in nginx \- Stack Overflow, consulté le septembre 26, 2025, [https://stackoverflow.com/questions/25778420/fail2ban-regular-to-find-403-request-in-nginx](https://stackoverflow.com/questions/25778420/fail2ban-regular-to-find-403-request-in-nginx)  
26. Fail2Ban+Nginx (blocking repeated 404's, etc) \- Such geek. Wow., consulté le septembre 26, 2025, [https://www.ericlight.com/fail2bannginx-blocking-repeated-404s-etc.html](https://www.ericlight.com/fail2bannginx-blocking-repeated-404s-etc.html)  
27. brute force \- fail2ban 404 bruteforcing sharex \- Super User, consulté le septembre 26, 2025, [https://superuser.com/questions/1472595/fail2ban-404-bruteforcing-sharex](https://superuser.com/questions/1472595/fail2ban-404-bruteforcing-sharex)  
28. Fail2Ban ignore 404 of local redirect \- apache \- Stack Overflow, consulté le septembre 26, 2025, [https://stackoverflow.com/questions/66000956/fail2ban-ignore-404-of-local-redirect](https://stackoverflow.com/questions/66000956/fail2ban-ignore-404-of-local-redirect)  
29. Protect Nginx Auth with Fail2Ban \- Ariq Naufal, consulté le septembre 26, 2025, [https://ariq.nauf.al/blog/protect-nginx-auth-with-fail2ban/](https://ariq.nauf.al/blog/protect-nginx-auth-with-fail2ban/)  
30. WP fail2ban – Advanced Security – WordPress plugin, consulté le septembre 26, 2025, [https://wordpress.org/plugins/wp-fail2ban/](https://wordpress.org/plugins/wp-fail2ban/)  
31. Fail2ban for WordPress on DigitalOcean \- Sail Blog, consulté le septembre 26, 2025, [https://blog.sailed.io/fail2ban-wordpress-digitalocean/](https://blog.sailed.io/fail2ban-wordpress-digitalocean/)  
32. Fail2ban WordPress the easy way \- Community Support \- Hestia Control Panel \- Discourse, consulté le septembre 26, 2025, [https://forum.hestiacp.com/t/fail2ban-wordpress-the-easy-way/5609](https://forum.hestiacp.com/t/fail2ban-wordpress-the-easy-way/5609)  
33. How to set-up fail2ban for a WordPress site \- Dogsbody Technology, consulté le septembre 26, 2025, [https://www.dogsbody.com/blog/how-to-set-up-fail2ban-for-a-wordpress-site/](https://www.dogsbody.com/blog/how-to-set-up-fail2ban-for-a-wordpress-site/)  
34. Basic Wordpress Fail2Ban Filter (Debian/Ubuntu Apache2) \- GitHub Gist, consulté le septembre 26, 2025, [https://gist.github.com/welenofsky/f2fad00da57cd11c17d6be20680e5286](https://gist.github.com/welenofsky/f2fad00da57cd11c17d6be20680e5286)  
35. How To Use Fail2ban With WordPress And Cloudflare Proxy, consulté le septembre 26, 2025, [https://runcloud.io/blog/fail2ban-wordpress-cloudflare](https://runcloud.io/blog/fail2ban-wordpress-cloudflare)  
36. fail2ban-client \- configure and control the server \- Ubuntu Manpage, consulté le septembre 26, 2025, [https://manpages.ubuntu.com/manpages/focal/man1/fail2ban-client.1.html](https://manpages.ubuntu.com/manpages/focal/man1/fail2ban-client.1.html)  
37. fail2ban-client \- configure and control the server \- Ubuntu Manpage, consulté le septembre 26, 2025, [https://manpages.ubuntu.com/manpages/xenial/man1/fail2ban-client.1.html](https://manpages.ubuntu.com/manpages/xenial/man1/fail2ban-client.1.html)  
38. How To Install Fail2ban On Ubuntu \- UpCloud, consulté le septembre 26, 2025, [https://upcloud.com/resources/tutorials/install-fail2ban-ubuntu/](https://upcloud.com/resources/tutorials/install-fail2ban-ubuntu/)  
39. How to not unban everyone when the service stops · fail2ban fail2ban · Discussion \#3009 \- GitHub, consulté le septembre 26, 2025, [https://github.com/fail2ban/fail2ban/discussions/3009](https://github.com/fail2ban/fail2ban/discussions/3009)  
40. Comment configurer Fail2Ban sur AlmaLinux 9 \- Shapehost, consulté le septembre 26, 2025, [https://shape.host/resources/comment-configurer-fail2ban-sur-almalinux-9](https://shape.host/resources/comment-configurer-fail2ban-sur-almalinux-9)  
41. Fail2Ban Commands \- VitalPBX Wiki, consulté le septembre 26, 2025, [https://wiki.vitalpbx.com/wiki/faq/fail2ban-commands/](https://wiki.vitalpbx.com/wiki/faq/fail2ban-commands/)  
42. Fail2ban : comment débannir une ou plusieurs adresses IP ? | IT ..., consulté le septembre 26, 2025, [https://www.it-connect.fr/fail2ban-comment-debannir-une-ou-plusieurs-adresses-ip/](https://www.it-connect.fr/fail2ban-comment-debannir-une-ou-plusieurs-adresses-ip/)  
43. Fail2ban \- lister et trier les IP bannies \- Le Blog de Christophe CUCCIARDI, consulté le septembre 26, 2025, [https://christophe.cucciardi.fr/fail2ban-lister-et-trier-les-ip-bannies/](https://christophe.cucciardi.fr/fail2ban-lister-et-trier-les-ip-bannies/)  
44. fail2ban-client: configure and control the server | fail2ban-server ..., consulté le septembre 26, 2025, [https://www.mankier.com/1/fail2ban-client](https://www.mankier.com/1/fail2ban-client)  
45. fail2ban-client(1) \- FreeBSD Manual Pages, consulté le septembre 26, 2025, [https://man.freebsd.org/cgi/man.cgi?query=fail2ban-client\&apropos=0\&sektion=1\&manpath=FreeBSD+13.2-RELEASE+and+Ports\&arch=default\&format=html](https://man.freebsd.org/cgi/man.cgi?query=fail2ban-client&apropos=0&sektion=1&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html)  
46. Howto ban IP with Fail2Ban manually by command line? \- Stack Overflow, consulté le septembre 26, 2025, [https://stackoverflow.com/questions/29018312/howto-ban-ip-with-fail2ban-manually-by-command-line](https://stackoverflow.com/questions/29018312/howto-ban-ip-with-fail2ban-manually-by-command-line)  
47. How to Unban an IP properly with Fail2Ban \- Server Fault, consulté le septembre 26, 2025, [https://serverfault.com/questions/285256/how-to-unban-an-ip-properly-with-fail2ban](https://serverfault.com/questions/285256/how-to-unban-an-ip-properly-with-fail2ban)  
48. Fail2ban : compter et classer les IP bannies \- Cybernaute.ch, consulté le septembre 26, 2025, [https://www.cybernaute.ch/fail2ban-compter-et-classer-les-ip-bannies/](https://www.cybernaute.ch/fail2ban-compter-et-classer-les-ip-bannies/)  
49. How to show all banned IP with fail2ban? \- Server Fault, consulté le septembre 26, 2025, [https://serverfault.com/questions/841183/how-to-show-all-banned-ip-with-fail2ban](https://serverfault.com/questions/841183/how-to-show-all-banned-ip-with-fail2ban)  
50. Attaque Brute Force : Comment Hydra craque les mots de passe ?, consulté le septembre 26, 2025, [https://datascientest.com/bruteforce-hydra-tout-savoir](https://datascientest.com/bruteforce-hydra-tout-savoir)  
51. Débannir une IP bannie via iptables par Fail2ban | Commandes et ..., consulté le septembre 26, 2025, [https://www.it-connect.fr/debannir-une-ip-bannie-via-iptables-par-fail2ban/](https://www.it-connect.fr/debannir-une-ip-bannie-via-iptables-par-fail2ban/)  
52. fail2ban and nftables – Useful Tips \- softworx.at, consulté le septembre 26, 2025, [https://www.softworx.at/en/fail2ban-and-nftables-useful-tips/](https://www.softworx.at/en/fail2ban-and-nftables-useful-tips/)  
53. please add counter to nftables block rule, so you can easily see if it works \#2767 \- GitHub, consulté le septembre 26, 2025, [https://github.com/fail2ban/fail2ban/issues/2767](https://github.com/fail2ban/fail2ban/issues/2767)  
54. Scan de vulnérabilités web avec Nikto sous Kali | LabEx, consulté le septembre 26, 2025, [https://labex.io/fr/tutorials/kali-kali-vulnerability-scanning-with-nikto-552301](https://labex.io/fr/tutorials/kali-kali-vulnerability-scanning-with-nikto-552301)  
55. How to Scan for Vulnerabilities on Any Website Using Nikto \- Null Byte \- WonderHowTo, consulté le septembre 26, 2025, [https://null-byte.wonderhowto.com/how-to/scan-for-vulnerabilities-any-website-using-nikto-0151729/](https://null-byte.wonderhowto.com/how-to/scan-for-vulnerabilities-any-website-using-nikto-0151729/)  
56. Nikto vulnerability scanner: Complete guide \- Hackercool Magazine, consulté le septembre 26, 2025, [https://www.hackercoolmagazine.com/nikto-vulnerability-scanner-complete-guide/](https://www.hackercoolmagazine.com/nikto-vulnerability-scanner-complete-guide/)  
57. How to Scan Vulnerabilities of Websites using Nikto in Linux? \- GeeksforGeeks, consulté le septembre 26, 2025, [https://www.geeksforgeeks.org/linux-unix/how-to-scan-vulnerabilities-of-websites-using-nikto-in-linux/](https://www.geeksforgeeks.org/linux-unix/how-to-scan-vulnerabilities-of-websites-using-nikto-in-linux/)  
58. Scan for Vulnerabilities on Any Website and Web Servers using Nikto | Kali Linux and Metasploitable \- YouTube, consulté le septembre 26, 2025, [https://www.youtube.com/watch?v=U0clIiWWwXM](https://www.youtube.com/watch?v=U0clIiWWwXM)