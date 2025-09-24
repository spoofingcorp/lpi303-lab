# 🛡️ Déploiement d'un WAF ModSecurity de niveau production avec Apache sur Ubuntu 24.04

## **Partie I : Le guide de laboratoire définitif pour le déploiement de ModSecurity et de l'OWASP CRS**

Ce guide fournit une méthodologie complète et éprouvée pour la mise en place d'un pare-feu applicatif web (**WAF**) robuste basé sur **ModSecurity** et l'**OWASP Core Rule Set (CRS)** sur un serveur web Apache fonctionnant sous Ubuntu 24.04. Chaque étape est conçue pour construire une couche de défense sécurisée, maintenable et prête pour la production.

### **1.0 Section 1 : Préparation de l'environnement de base**

Avant de déployer des contrôles de sécurité avancés, il est impératif de s'assurer que le système d'exploitation et les services sous-jacents sont correctement configurés et sécurisés.

#### **1.1 Provisionnement du système et sécurité de base sur Ubuntu 24.04**

La procédure commence avec une instance de serveur Ubuntu 24.04 fraîchement installée. La première action consiste à mettre à jour l'index des paquets et à appliquer toutes les mises à jour.

```bash
sudo apt update && sudo apt upgrade -y
```

💡 **Bonne pratique** : Pour adhérer au principe du moindre privilège, toutes les opérations suivantes doivent être effectuées par un **utilisateur non-root disposant de privilèges `sudo`**.

#### **1.2 Installation et gestion du service Apache2**

Le serveur web Apache est disponible dans les dépôts par défaut d'Ubuntu, garantissant une installation simple.

```bash
sudo apt install apache2 -y
```

Une fois l'installation terminée, vérifiez la version et l'état du service.

  * **Vérifier la version d'Apache**
    ```bash
    apache2 -v
    ```
  * **Vérifier l'état du service Apache**
    ```bash
    sudo systemctl status apache2
    ```

Configurez Apache pour qu'il démarre automatiquement au démarrage du système.

```bash
sudo systemctl enable apache2
```

#### **1.3 Configuration du pare-feu avec UFW pour les services web**

La configuration d'un pare-feu au niveau de l'hôte est une étape de sécurité non négociable. **UFW (Uncomplicated Firewall)** fournit une interface conviviale pour gérer `iptables`.

  * **Lister les profils d'application disponibles**
    ```bash
    sudo ufw app list
    ```
  * **Autoriser le trafic web standard (`HTTP` & `HTTPS`)**
    ```bash
    sudo ufw allow 'Apache Full'
    ```
  * **Activer le pare-feu**
    ```bash
    sudo ufw enable
    ```
  * **Vérifier l'état et les règles actives**
    ```bash
    sudo ufw status
    ```

Cette configuration garantit que seules les connexions **SSH** (port 22) et **web** (ports 80 et 443) sont autorisées.

-----

### **2.0 Section 2 : Installation et configuration du moteur ModSecurity**

Cette section se concentre sur l'installation et la configuration du module **ModSecurity**, qui constitue le moteur d'inspection du WAF.

#### **2.1 Déploiement du module `libapache2-mod-security2`**

Installez le module depuis les dépôts officiels d'Ubuntu.

```bash
sudo apt install libapache2-mod-security2 -y
```

Le processus d'installation active généralement le module. Vérifiez et activez-le manuellement si nécessaire.

```bash
sudo a2enmod security2
```

Un redémarrage d'Apache est requis pour charger le nouveau module.

```bash
sudo systemctl restart apache2
```

#### **2.2 Établissement de la configuration de base `modsecurity.conf`**

La meilleure pratique consiste à copier le fichier de configuration modèle pour créer la configuration active.

```bash
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```

Le fichier `/etc/modsecurity/modsecurity.conf` devient le fichier de configuration principal du moteur ModSecurity.

#### **2.3 Examen approfondi des directives essentielles**

Plusieurs directives dans `modsecurity.conf` sont d'une importance capitale :

  * **`SecRuleEngine`**: C'est l'interrupteur principal du WAF.

      * `On` : Les règles sont actives et bloquent les menaces.
      * `Off` : Le module est complètement désactivé.
      * `DetectionOnly` : Les règles journalisent les menaces mais ne bloquent rien.
        ⚠️ **Crucial** : Pour une nouvelle installation, commencez toujours en mode **`DetectionOnly`** pour identifier les faux positifs sans impacter les utilisateurs.

  * **`SecAuditEngine`**: Contrôle la journalisation d'audit. La valeur `On` est recommandée au début pour enregistrer toutes les transactions.

  * **`SecAuditLog`**: Définit le chemin du fichier journal d'audit, généralement `/var/log/apache2/modsec_audit.log`.

  * **`SecRequestBodyAccess On`**: Directive **critique**. Elle ordonne à ModSecurity d'inspecter le corps des requêtes (données de formulaire POST, JSON, etc.), où se trouvent la plupart des attaques applicatives.

  * **`SecRequestBodyLimit`**: Une mesure de défense contre les attaques par déni de service (**DoS**) qui limite la taille maximale du corps d'une requête.

-----

### **3.0 Section 3 : Intégration de l'OWASP Core Rule Set (CRS)**

Cette section détaille l'intégration de l'intelligence qui alimente le moteur ModSecurity.

#### **3.1 Le rôle du CRS en tant que couche d'intelligence du WAF**

ModSecurity est le **moteur**, mais le CRS est le **"cerveau"**. C'est cet ensemble de règles qui contient les signatures et les heuristiques pour détecter les attaques connues (injections SQL, XSS, etc.).

#### **3.2 Acquisition et structuration de la dernière version stable du CRS**

⚠️ **Important** : Le paysage des menaces évolue. Utilisez toujours la dernière version stable du CRS depuis le dépôt GitHub officiel `coreruleset`.

1.  **Télécharger la dernière version stable** (Vérifiez la version la plus récente sur la [page des publications du CRS](https://github.com/coreruleset/coreruleset/releases)).
    ```bash
    # Remplacez vX.Y.Z par la dernière version
    VERSION="v4.18.0" 
    cd /tmp
    wget "https://github.com/coreruleset/coreruleset/archive/refs/tags/${VERSION}.tar.gz"
    ```
2.  **Extraire et organiser les fichiers**
    ```bash
    tar -xzvf ${VERSION}.tar.gz
    sudo mv coreruleset-${VERSION/v/} /etc/apache2/modsecurity-crs
    ```

#### **3.3 Configuration de `crs-setup.conf` : Niveaux de paranoïa**

Le CRS est livré avec un fichier de configuration modèle qui doit être activé et personnalisé.

```bash
cd /etc/apache2/modsecurity-crs
sudo cp crs-setup.conf.example crs-setup.conf
```

Le fichier `crs-setup.conf` permet de définir les **Niveaux de Paranoïa (Paranoia Levels - PL)**, qui offrent un compromis entre sécurité et risque de faux positifs.

| Niveau de Paranoïa | Description                                                                                              | Cas d'utilisation typique                                       | Risque de faux positifs |
| :----------------- | :------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------- | :---------------------- |
| **PL1 (Défaut)** | Protection de base contre les attaques les plus courantes.                                               | Tous les sites web et API. **Le point de départ recommandé.** | Faible                  |
| **PL2** | Sécurité accrue contre des attaques plus avancées.                                                       | Sites e-commerce, applications avec données sensibles.          | Moyen                   |
| **PL3** | Sécurité complète avec des règles plus restrictives.                                                     | Applications à haute sécurité (gouvernement, finance).          | Élevé                   |
| **PL4** | Niveau paranoïaque. Extrêmement restrictif pour une sécurité maximale.                                    | API ou segments d'application avec un trafic très prévisible.   | Très élevé              |

Pour commencer, il est recommandé de définir le niveau de paranoïa à **1** dans `crs-setup.conf`.

Cherchez la section suivante (autour de la ligne 177 selon la version), et décommenter

```
# Uncomment this rule to change the default:
#
SecAction \
    "id:900000,\
    phase:1,\
    pass,\
    t:none,\
    nolog,\
    tag:'OWASP_CRS',\
    ver:'OWASP_CRS/4.18.0',\
    setvar:tx.blocking_paranoia_level=1"
```

Différence avec SecDefaultAction

- SecDefaultAction : Définit l’action par défaut appliquée à chaque règle du CRS (ex. loguer, bloquer, laisser passer).
- phase:1 → analyse en début de requête
- log,auditlog → écrit dans les logs et l’audit log
- pass → ne bloque pas automatiquement à ce stade (c’est le mode Anomaly Scoring qui décide plus tard).
- SecAction setvar:tx.paranoia_level=1 : Définit le niveau de paranoïa (= quelles règles seront chargées).


#### **3.4 Activation du CRS dans la configuration d'Apache**

Indiquez à Apache de charger les fichiers du CRS. Ajoutez les lignes suivantes à la fin de `/etc/apache2/mods-enabled/security2.conf`.

```
<IfModule security2_module>
        # Default Debian dir for modsecurity's persistent data
        SecDataDir /var/cache/modsecurity


        # Inclure la configuration du CRS (DOIT ÊTRE EN PREMIER)
        IncludeOptional /etc/apache2/modsecurity-crs/crs-setup.conf
        # Inclure les fichiers de règles du CRS
        IncludeOptional /etc/apache2/modsecurity-crs/rules/*.conf

        # Include all the *.conf files in /etc/modsecurity.
        # Keeping your local configuration in that directory
        # will allow for an easy upgrade of THIS file and
        # make your life easier
#        IncludeOptional /etc/modsecurity/*.conf     #### COMMENTER

        # Include OWASP ModSecurity CRS rules if installed
#       IncludeOptional /usr/share/modsecurity-crs/*.load    #### COMMENTER
</IfModule>
```

-----

### Activer ModSecurity sur le default site Apache

/etc/apache2/sites-enabled/000-default.conf      

```
<VirtualHost *:80>
        #ServerName www.example.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        SecRuleEngine On   #### AJOUTER CETTE OPTION

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
#### Désactiver la RULES bloqué l'accès via IP

Si vous faites un curl apprésent, le site ne sera pas accesible car nous n'avons pas de noms de domaine.

ModSecurity: Warning. Pattern match "(?:^([\\\\d.]+|\\\\[[\\\\da-f:]+\\\\]|[\\\\da-f:]+)(:[\\\\d]+)?$)" at REQUEST_HEADERS:Host. [file "/etc/apache2/modsecurity-crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf"] [line "728"] [id "920350"] [msg "Host header is a numeric IP address"] [data "192.168.20.180"] [severity "WARNING"] 

#### Editer la Rules REQUEST-920-PROTOCOL-ENFORCEMENT.conf (vers la ligne 714) pour pass Host header is a numeric IP address

`nano /etc/apache2/modsecurity-crs/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf -c`

```
SecRule REQUEST_HEADERS:Host "@rx (?:^([\d.]+|\[[\da-f:]+\]|[\da-f:]+)(:[\d]+)?$)" \
"id:920350,\
    phase:1,\
    pass,\  ### PASSER de block à pass
    t:none,\
    msg:'Host header is a numeric IP address',\
    logdata:'%{MATCHED_VAR}',\
    tag:'application-multi',\
    tag:'language-multi',\
    tag:'platform-multi',\
    tag:'attack-protocol',\
    tag:'paranoia-level/1',\
    tag:'OWASP_CRS',\
    tag:'OWASP_CRS/PROTOCOL-ENFORCEMENT',\
    tag:'capec/1000/210/272',\
    ver:'OWASP_CRS/4.18.0',\
    severity:'WARNING',\
    setvar:'tx.inbound_anomaly_score_pl1=+%{tx.warning_anomaly_score}'"
```

### **4.0 Section 4 : Vérification du système et simulation d'attaques**

Validez que la configuration est correcte et que le WAF fonctionne comme prévu.

#### **4.1 Vérification de la syntaxe et redémarrage**

Avant de redémarrer, vérifiez toujours la syntaxe de la configuration Apache pour éviter les pannes.

```bash
sudo apache2ctl configtest
```

Si la commande retourne `Syntax OK`, redémarrez le service.

```bash
sudo systemctl restart apache2
```

#### **4.2 🧪 Simulation de vecteurs d'attaque avec `curl`**

Envoyez des requêtes malveillantes simples pour vérifier que le WAF les intercepte. Une réponse **`403 Forbidden`** est attendue.

  * **Test de Path Traversal**
    ```bash
    curl -i "http://<votre_ip_serveur>/?exec=/etc/passwd"
    ```
  * **Test de Cross-Site Scripting (XSS)**
    ```bash
    curl -i "http://<votre_ip_serveur>/?search=<script>alert('xss')</script>"
    ```
  * **Test d'injection SQL (SQLi)**
    ```bash
    curl -i "http://<votre_ip_serveur>/?id=1' OR 1=1--"
    ```

#### **4.3 Analyse des réponses `403 Forbidden`**

Recevoir un code `403 Forbidden` confirme que ModSecurity et l'OWASP CRS ont correctement identifié et bloqué la requête malveillante.

-----

### **5.0 Section 5 : Maîtrise de la journalisation et de l'analyse des alertes**

Un WAF qui bloque silencieusement est difficile à gérer. Comprendre ses journaux est une compétence essentielle.

#### **5.1 Navigation dans les journaux Apache et ModSecurity**

Deux fichiers journaux principaux sont à surveiller :

1.  **Journal d'erreurs d'Apache (`/var/log/apache2/error.log`)** : C'est le **signal**. Il vous alerte qu'un événement s'est produit avec un message concis et un ID de transaction unique.
    ```bash
    sudo tail -f /var/log/apache2/error.log
    ```
2.  **Journal d'audit de ModSecurity (`/var/log/apache2/modsec_audit.log`)** : C'est le **contexte**. Il contient les détails complets (forensiques) de chaque transaction.
    ```bash
    sudo tail -f /var/log/apache2/modsec_audit.log
    ```

💡 **Flux de travail** : Repérez l'alerte dans `error.log`, copiez l'ID unique, puis utilisez `grep` pour trouver l'entrée complète dans `modsec_audit.log` pour une analyse approfondie.

#### **5.2 Anatomie d'une entrée du journal `modsec_audit.log`**

Chaque entrée est composée de plusieurs sections identifiées par une lettre.

| Section | Nom                    | Description                                                                          |
| :------ | :--------------------- | :----------------------------------------------------------------------------------- |
| **A** | En-tête d'audit        | Métadonnées : Horodatage, ID unique, IP source/destination.                            |
| **B** | En-têtes de la requête | Liste complète des en-têtes HTTP envoyés par le client.                                |
| **C** | Corps de la requête    | La charge utile (ex: données POST). Crucial pour l'analyse.                            |
| **H** | Pied de page d'audit   | Informations récapitulatives, y compris les messages des règles déclenchées.         |
| **K** | Règles correspondantes | Liste consolidée de toutes les règles qui ont correspondu. Essentiel pour le diagnostic. |
| **Z** | Délimiteur final       | Marque la fin de l'entrée du journal.                                                  |

-----

### **6.0 Section 6 : Réglage de base et gestion des faux positifs**

Le **réglage (tuning)** est le processus d'adaptation du WAF au comportement normal de votre application pour minimiser les faux positifs.

#### **6.1 La stratégie de déploiement initial en mode `DetectionOnly`**

La pratique la plus importante est de commencer avec `SecRuleEngine DetectionOnly`. Cela permet de collecter des données sur les blocages potentiels sans affecter le trafic légitime.

#### **6.2 Création d'un fichier d'exclusion de règles personnalisé**

⚠️ **Ne modifiez jamais directement les fichiers de règles du CRS**. Créez plutôt un fichier d'exclusion personnalisé qui sera chargé avant les règles principales.

```bash
sudo touch /etc/apache2/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

#### **6.3 Bonnes pratiques pour la désactivation de règles via `SecRuleRemoveById`**

1.  **Déployer en mode `DetectionOnly`**.
2.  **Surveiller les journaux** pendant une période représentative (quelques jours à une semaine).
3.  **Identifier les faux positifs**. Repérez l'ID de la règle qui se déclenche sur du trafic légitime (ex: `942100`).
4.  **Créer une exclusion spécifique et documentée**. Ajoutez une directive dans votre fichier d'exclusion :
    ```apache
    # Fichier : /etc/apache2/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf

    # Désactivation de la règle 942100 pour l'application MyApp
    # Raison : Faux positif sur le champ 'description' qui autorise les apostrophes.
    # Date : 2025-10-27
    # Analyste : J. Doe
    SecRuleRemoveById 942100
    ```
5.  **Tester et valider**. Redémarrez Apache et confirmez que le trafic légitime passe et que les attaques sont toujours bloquées.
6.  **Passer en mode blocage**. Une fois que les journaux sont "propres", vous pouvez passer en toute confiance à `SecRuleEngine On` dans `modsecurity.conf`.

-----

## **Partie III : Conclusion et étapes suivantes**

### **7.0 Résumé des bonnes pratiques et voie vers le réglage avancé**

La mise en place d'un WAF efficace est un processus méthodique. Les points clés à retenir sont :

  * ✅ **Moteur vs Règles** : ModSecurity est le moteur, l'OWASP CRS est l'intelligence.
  * ✅ **Source fiable** : Utilisez toujours la dernière version stable du CRS depuis son dépôt officiel.
  * ✅ **`DetectionOnly` d'abord** : Une phase d'observation est non négociable.
  * ✅ **Analyse des journaux** : Utilisez `error.log` comme signal et `modsec_audit.log` comme contexte.
  * ✅ **Réglage structuré** : Gérez les faux positifs via des fichiers d'exclusion personnalisés.

Ce guide a couvert les fondations essentielles. Les étapes suivantes incluent l'écriture de règles personnalisées, la création d'exclusions plus granulaires (`SecRuleUpdateTargetById`) et l'intégration des journaux dans un système **SIEM** (Security Information and Event Management) pour une surveillance centralisée.

### **Sources des citations**

1.  Apache2 : installer le ModSecurity (WAF) - RDR-IT
2.  Notes on installing ModSecurity and applying it to Apache (Ubuntu 24.04 LTS)
3.  How To Configure Apache Security on Ubuntu & Debian – TecAdmin
4.  How to Install the ModSecurity Apache Module - InMotion Hosting
5.  How to Install Apache Web Server on Ubuntu 24.04 - Vultr Docs
6.  How to use Apache2 modules - Ubuntu Server documentation
7.  owasp-modsecurity/ModSecurity - GitHub
8.  coreruleset/coreruleset: OWASP CRS (Official Repository) - GitHub
9.  OWASP CRS Project - Releases
10. OWASP CRS Dev Guide
11. How to Install and Configure ModSecurity on Apache for Ubuntu
12. Comprehensive Guide to ModSecurity: Logs, Configuration, and Important Limits
13. ModSecurity Handbook: Getting Started: Chapter 4. Logging - Feisty Duck
14. Upgrading the provided OWASP Core Rule Set of ModSecurity - Nevis documentation
15. OWASP ModSecurity Core Rule Set (CRS) Project (Official Repository) - GitHub (SpiderLabs - Ancien)
16. CRS versions 4.8.0 and 3.3.7 released - Coreruleset.org
17. Security Overview · coreruleset/coreruleset - GitHub
18. Releases · coreruleset/coreruleset - GitHub
19. OWASP CRS Project
20. NGINX + ModSecurity v3 + OWASP CRS on Ubuntu 24.04 LTS – Step by Step – Part 2
21. CRS Installation :: CRS Documentation - OWASP CRS Project
22. mod security - Test whether mod\_security is actually working - Server Fault
23. ModSecurity: Logging and Debugging - F5
