# Splunk Enterprise Administrator Bootcamp v9.4 — Synthèse Lab Guide

> Source : Enterprise Administrator Bootcamp Lab Exercises v9.4 (2026)  
> 21 modules de lab · 184 pages

---

## Environnement de lab — rappel

| Serveur | Rôle | Accès SSH |
|---------|------|-----------|
| **ds1** | Deployment/Test Server (Splunk Enterprise admin) | `ssh student@{PUBLIC IP}` |
| **sh1** | Search Head (user : student / rôle power) | `ssh sh1` |
| **hf1** | Heavy Forwarder 1 | `ssh hf1` |
| **uf1** | Universal Forwarder 1 | `ssh uf1` |
| **uf2** | Universal Forwarder 2 | `ssh uf2` |
| **idx1** | Indexer 1 | `ssh idx1` |
| **idx2** | Indexer 2 | `ssh idx2` |

**SPLUNK_HOME :**
- Indexers / ds1 / sh1 / hf1 → `/opt/splunk`
- Forwarders UF → `/opt/splunkforwarder`

---

## Sommaire

1. [Lab 1 — Configure a Splunk Server](#lab-1--configure-a-splunk-server)
2. [Lab 2 — Monitor Splunk](#lab-2--monitor-splunk)
3. [Lab 3 — License Splunk](#lab-3--license-splunk)
4. [Lab 4 — Use Configuration Files](#lab-4--use-configuration-files)
5. [Lab 5 — Use Apps](#lab-5--use-apps)
6. [Lab 6 — Create Indexes](#lab-6--create-indexes)
7. [Lab 7 — Manage Indexes](#lab-7--manage-indexes)
8. [Lab 8 — Manage Users and Roles](#lab-8--manage-users-and-roles)
9. [Lab 9 — Add a Local Data Input](#lab-9--add-a-local-data-input)
10. [Lab 10 — Configure Forwarders](#lab-10--configure-forwarders)
11. [Lab 11 — Customize Forwarders](#lab-11--customize-forwarders)
12. [Lab 12 — Manage Forwarders (Deployment Server)](#lab-12--manage-forwarders)
13. [Lab 13 — Monitor Inputs](#lab-13--monitor-inputs)
14. [Lab 14 — Network Inputs](#lab-14--network-inputs)
15. [Lab 15 — Scripted Inputs](#lab-15--scripted-inputs)
16. [Lab 16 — Agentless Inputs (HEC)](#lab-16--agentless-inputs-hec)
17. [Lab 17 — Fine-tune Inputs](#lab-17--fine-tune-inputs)
18. [Lab 18 — Create a New Source Type](#lab-18--create-a-new-source-type)
19. [Lab 19 — Manipulating Input Data](#lab-19--manipulating-input-data)
20. [Lab 20 — Routing Input Data](#lab-20--routing-input-data)
21. [Lab 21 — Support Knowledge Objects](#lab-21--support-knowledge-objects)
22. [Référence complète des commandes CLI](#référence-complète-des-commandes-cli)
23. [Fichiers de configuration produits dans les labs](#fichiers-de-configuration-produits-dans-les-labs)
24. [Commandes de troubleshooting](#commandes-de-troubleshooting)

---

## Lab 1 — Configure a Splunk Server

### Objectifs
- Accéder à Splunk Web et modifier les paramètres de base
- Utiliser le CLI pour récupérer les informations système
- Observer les changements dans server.conf

### Commandes CLI utilisées

```bash
cd /opt/splunk/bin

./splunk status                        # Statut des processus splunkd
./splunk version                       # Version et build number
./splunk show web-port                 # Port Splunk Web (8000)
./splunk show splunkd-port             # Port splunkd (8089)
./splunk show appserver-ports          # Ports app-server (8065)
./splunk show kvstore-port             # Port KV Store (8191)
./splunk show servername               # Nom du serveur
./splunk show default-hostname         # Hostname par défaut pour les inputs

./splunk set servername splunk01       # Changer le nom du serveur
./splunk restart                       # Redémarrer après changement
./splunk login                         # S'authentifier pour les commandes suivantes
```

### Fichiers observés

```bash
# Vérifier la configuration en mémoire vs sur disque
./splunk btool server list general
./splunk btool server list general --debug       # Affiche le fichier source

./splunk show config server                      # Config en mémoire
./splunk show config server | grep serverName    # Filtrer un paramètre
```

### Configuration Change Tracker

```spl
# Rechercher tous les changements récents
index=_configtracker

# Filtrer sur un fichier et paramètre spécifique
index=_configtracker server.conf serverName

# Tableau des changements de serverName
index=_configtracker server.conf serverName data.action=update
| table _time data.changes{}.properties{}.old_value data.changes{}.properties{}.new_value
```

> ⚠️ **Différence btool vs show config :** `btool` lit les fichiers sur disque. `show config` lit la configuration en mémoire. Un changement manuel dans un .conf apparaît dans `btool` **mais pas** dans `show config` tant que Splunk n'a pas été redémarré.

---

## Lab 2 — Monitor Splunk

### Objectifs
- Explorer le Health Report et le Health Report Manager
- Activer et utiliser le Monitoring Console (MC)
- Créer un fichier diag

### Commandes CLI

```bash
cd /opt/splunk/bin

# Générer un bundle de diagnostique
./splunk diag
# Résultat : /opt/splunk/diag-<hostname>-<date>.tar.gz

# Exemple de sortie :
# Splunk diagnosis file created: /opt/splunk/diag-aio1-2025-02-10_19-49-30.tar.gz
```

### Navigation MC (Settings → Monitoring Console)

- **Settings → General Setup** → vérifier les rôles découverts
- **Summary** → vue Deployment Topology et Deployment Metrics
- **Indexing → Performance → Indexing Performance: Instance** → pipeline fill ratio
- **Health Check** → lancer `Start` pour vérifier l'état
- **Settings → Health Check Items** → activer/désactiver des vérifications

---

## Lab 3 — License Splunk

### Objectifs
- Ajouter une licence Enterprise
- Modifier un pool de licence
- Activer une alerte MC sur l'usage des licences

### Navigation

1. **Settings → Licensing** → Add license → choisir le fichier `.license`
2. Modifier le pool : cliquer Edit sur `auto_generated_pool_enterprise`
3. Alerte MC : Settings → Monitoring Console → Alerts → **DMC Alert - Total License Usage Near Daily Quota**

> ⚠️ Après ajout d'une licence, **redémarrer Splunk** (Messages → Click here to restart).

---

## Lab 4 — Use Configuration Files

### Objectifs
- Explorer la structure des répertoires de conf
- Comprendre le layering
- Utiliser btool pour déboguer

### Navigation fichiers

```bash
cd /opt/splunk/etc/system

ls README        # Documentation : *.conf.spec et *.conf.example
ls default       # Valeurs par défaut Splunk (ne pas modifier)
ls local         # Modifications locales système

# Lire la spec d'un paramètre (ex. serverName à la ligne 39)
more +39 README/server.conf.spec

# Voir la valeur par défaut
more +16 default/server.conf

# Voir la valeur locale (overridée)
more local/server.conf
```

### Commandes btool essentielles

```bash
cd /opt/splunk/bin

# Voir la config fusionnée d'une stanza
./splunk btool server list general

# Voir d'où vient chaque paramètre
./splunk btool server list general --debug

# Exemple de sortie :
# /opt/splunk/etc/system/default/server.conf   python.version = force_python3
# /opt/splunk/etc/system/local/server.conf     serverName = splunk01

# Vérifier le config change tracker
./splunk btool server list config_change_tracker

# Config en mémoire
./splunk show config server
./splunk show config server | grep serverName
```

### Cycle de vie d'un changement de configuration

```
1. Modifier le .conf sur disque
2. btool voit le nouveau paramètre
3. show config voit encore l'ancien (en mémoire)
4. ./splunk restart
5. btool ET show config voient la même valeur
```

---

## Lab 5 — Use Apps

### Objectifs
- Installer une app depuis un fichier .spl
- Modifier les permissions d'une app
- Vérifier l'installation via les fichiers

### Commandes

```bash
cd /opt/splunk/etc/apps

ls                                              # Lister les apps installées
ls system_admin_app                             # Voir le contenu d'une app
more system_admin_app/default/app.conf          # Propriétés de l'app
more system_admin_app/metadata/local.meta       # Permissions
```

### Permissions dans local.meta

```ini
[]
access = read : [ admin ], write : [ admin ]
export = none
```

### Résultat attendu après installation

```bash
# L'index websales apparaît dans Settings → Indexes
# L'app system_admin_app apparaît dans /opt/splunk/etc/apps/
```

---

## Lab 6 — Create Indexes

### Objectifs
- Créer des index (securityops, sales, itops, test)
- Ajouter un file monitor vers securityops
- Indexer le fichier diag dans main
- Observer les alertes de licence

### Index créés dans ce lab

| Index | Usage |
|-------|-------|
| `securityops` | Données sécurité |
| `sales` | Données commerciales |
| `itops` | IT Operations |
| `test` | Tests et validation |

### Recherches de vérification

```spl
# Vérifier l'indexation dans securityops
index=securityops

# Contenu du diag (toutes les sources)
index=main source=*diag* host=splunk01 | stats count by source

# Informations système du diag
index=main source=*systeminfo.txt "diag launched" host=splunk01

# Vérifier l'alerte de licence
index=* | stats count by index
```

---

## Lab 7 — Manage Indexes

### Objectifs
- Utiliser le MC pour surveiller les index
- Configurer une rétention temporelle sur securityops
- Configurer une rétention volumétrique sur itops

### Édition manuelle de indexes.conf

```bash
# Fichier à éditer
nano /opt/splunk/etc/apps/search/local/indexes.conf
```

### Rétention temporelle — securityops

```ini
[securityops]
coldPath = $SPLUNK_DB/securityops/colddb
homePath = $SPLUNK_DB/securityops/db
maxDataSize = auto_high_volume
maxTotalDataSizeMB = 512000
thawedPath = $SPLUNK_DB/securityops/thaweddb
maxHotSpanSecs = 86400          # Bucket hot = max 1 jour
frozenTimePeriodInSecs = 7776000  # Rétention = 90 jours
```

### Rétention volumétrique — itops avec volumes

```ini
[volume:one]
path = /home/student/one/
maxVolumeDataSizeMB = 40000     # Volume max = 40 GB

[volume:two]
path = /home/student/two/
maxVolumeDataSizeMB = 80000     # Volume max = 80 GB

[itops]
coldPath = volume:two/itops/colddb
homePath = volume:one/itops/db
maxTotalDataSizeMB = 512000
thawedPath = $SPLUNK_DB/itops/thaweddb
homePath.maxDataSizeMB = 30000  # hot/warm max 30 GB
coldPath.maxDataSizeMB = 60000  # cold max 60 GB
```

```bash
# Redémarrer après modification
/opt/splunk/bin/splunk restart
```

---

## Lab 8 — Manage Users and Roles

### Objectifs
- Modifier les rôles user, power, admin
- Créer un rôle soc_analyst
- Vérifier les accès

### Rôle soc_analyst créé

```
Héritage : power
Default app : search
Indexes : main, securityops, websales (hérités)
```

### Vérification

```spl
# Vérifier les index accessibles depuis sh1 (connecté comme emaxwell)
host=* | stats count by index
```

---

## Lab 9 — Add a Local Data Input

### Objectifs
- Identifier les composants Splunk via une recherche
- Configurer le MC sur ds1
- Indexer un fichier local dans l'index test

### Recherches de vérification

```spl
# Identifier les serveurs Splunk accessibles depuis sh1
index=_internal splunk_server=* | dedup splunk_server | table splunk_server

# Vérifier l'input créé
source="/opt/log/www3/access.log" host="splunk01" index="test" sourcetype="access_combined_wcookie"
```

---

## Lab 10 — Configure Forwarders

### Objectifs
- Configurer le port de réception sur ds1
- Installer et configurer UF1 et HF1 pour envoyer vers ds1
- Ajouter un monitor input sur UF1

### Étapes complètes — configuration UF1

```bash
# Sur ds1 — configurer le port de réception
cd /opt/splunk/bin
./splunk btool inputs list splunktcp --debug    # Vérifier avant
./splunk enable listen 9997                     # Activer le port
./splunk btool inputs list splunktcp --debug    # Vérifier après

# SSH vers UF1
ssh uf1
cd /opt/splunkforwarder/bin

# Démarrer UF1
./splunk start --accept-license

# Boot-start (pas besoin de sudo pour start/stop)
./splunk stop
sudo ./splunk enable boot-start -systemd-managed 0 -user student
./splunk start

# Configurer le forwarding vers ds1
./splunk add forward-server ds1:9997
./splunk list forward-server                   # Vérifier : Active forwards: ds1:9997

# Vérifier outputs.conf généré
./splunk btool outputs list tcpout:default-autolb-group --debug

# Ajouter un monitor input
./splunk add monitor /opt/log/www1/access.log -sourcetype access_combined_wcookie -index test
./splunk list monitor                          # Vérifier

# Voir inputs.conf généré
more ../etc/apps/search/local/inputs.conf

./splunk restart
exit
```

### Étapes complètes — configuration HF1

```bash
ssh hf1
cd /opt/splunk/bin
./splunk start --accept-license
./splunk stop
sudo ./splunk enable boot-start -systemd-managed 0 -user student
./splunk start
./splunk add forward-server ds1:9997
./splunk list forward-server
./splunk btool outputs list tcpout:default-autolb-group --debug
./splunk restart
exit
```

### Recherches de vérification (sur ds1)

```spl
# Données internes de UF1
index=_internal sourcetype=splunkd host=uf1

# Données internes de HF1
index=_internal sourcetype=splunkd host=hf1

# Données du monitor sur UF1
index=test host=uf1
```

---

## Lab 11 — Customize Forwarders

### Objectifs
- Reconfigurer UF1 et HF1 vers idx1 et idx2
- Configurer HF1 comme forwarder intermédiaire pour UF2

### Reconfigurer UF1 → idx1 + idx2

```bash
ssh uf1
cd /opt/splunkforwarder/bin

./splunk list forward-server                   # Vérifier : ds1:9997
./splunk remove forward-server ds1:9997        # Supprimer ds1
./splunk add forward-server idx1:9997          # Ajouter idx1
./splunk add forward-server idx2:9997          # Ajouter idx2
./splunk list forward-server                   # Vérifier : idx1 et idx2
./splunk btool outputs list tcpout:default-autolb-group --debug
./splunk restart
exit
```

### Reconfigurer HF1 → idx1 + idx2

```bash
ssh hf1
cd /opt/splunk/bin
./splunk remove forward-server ds1:9997
./splunk add forward-server idx1:9997
./splunk add forward-server idx2:9997
./splunk list forward-server
./splunk btool outputs list tcpout:default-autolb-group --debug
./splunk restart
```

### Configurer HF1 comme forwarder intermédiaire (réception depuis UF2)

```bash
# Sur HF1 — ouvrir un port de réception pour UF2
./splunk enable listen 9901
more /opt/splunk/etc/apps/search/local/inputs.conf
# → [splunktcp://9901] connection_host = ip
exit
```

### Configurer UF2 → HF1 (via port 9901)

```bash
ssh uf2
cd /opt/splunkforwarder/bin
./splunk start --accept-license
./splunk stop
sudo ./splunk enable boot-start -systemd-managed 0 -user student
./splunk start
./splunk add forward-server hf1:9901
./splunk list forward-server                   # Vérifier : hf1:9901
./splunk btool outputs list tcpout:default-autolb-group --debug
./splunk restart
exit
```

### Recherches de vérification (sur sh1 en tant que student)

```spl
# Données de HF1 → indexers
index=_internal sourcetype=splunkd host=hf1
# splunk_server doit montrer idx1 et idx2

# Données de UF1 → indexers
index=_internal sourcetype=splunkd host=uf1

# Données de UF2 → HF1 → indexers
index=_internal sourcetype=splunkd host=uf2
```

---

## Lab 12 — Manage Forwarders

### Objectifs
- Supprimer les configs de forwarding système (system/local)
- Créer les apps `uf_base` et `hf_base` dans deployment-apps
- Configurer UF2 et HF1 comme deployment clients
- Créer des server classes sur le DS

### Étape 1 — Supprimer les configs forwarding existantes

```bash
# Sur UF2
ssh uf2
cd /opt/splunkforwarder/bin
./splunk list forward-server
./splunk btool outputs list tcpout:default --debug
./splunk remove forward-server hf1:9901
more /opt/splunkforwarder/etc/system/local/outputs.conf
rm /opt/splunkforwarder/etc/system/local/outputs.conf   # Supprimer le fichier résiduel
exit

# Sur HF1
ssh hf1
cd /opt/splunk/bin
./splunk remove forward-server idx1:9997
./splunk remove forward-server idx2:9997
rm /opt/splunk/etc/system/local/outputs.conf
rm /opt/splunk/etc/apps/search/local/inputs.conf        # Supprimer le port 9901
./splunk btool inputs list splunktcp://9901 --debug      # Vérifier : pas de résultat
./splunk restart
exit
```

### Étape 2 — Créer l'app uf_base sur ds1

```bash
# Créer le répertoire
mkdir -p /opt/splunk/etc/deployment-apps/uf_base/local

# Créer outputs.conf
nano /opt/splunk/etc/deployment-apps/uf_base/local/outputs.conf
```

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout-server://hf1:9901]

[tcpout:default-autolb-group]
disabled = false
server = hf1:9901
```

```bash
# Créer deploymentclient.conf (réduit le polling à 30s)
nano /opt/splunk/etc/deployment-apps/uf_base/local/deploymentclient.conf
```

```ini
[deployment-client]
phoneHomeIntervalInSecs = 30
```

### Étape 3 — Créer l'app hf_base sur ds1

```bash
mkdir -p /opt/splunk/etc/deployment-apps/hf_base/local

# outputs.conf
nano /opt/splunk/etc/deployment-apps/hf_base/local/outputs.conf
```

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout-server://idx1:9997]
[tcpout-server://idx2:9997]

[tcpout:default-autolb-group]
disabled = false
server = idx1:9997,idx2:9997
```

```bash
# inputs.conf (port de réception HF)
nano /opt/splunk/etc/deployment-apps/hf_base/local/inputs.conf
```

```ini
[splunktcp://9901]
connection_host = ip
```

```bash
# deploymentclient.conf
nano /opt/splunk/etc/deployment-apps/hf_base/local/deploymentclient.conf
```

```ini
[deployment-client]
phoneHomeIntervalInSecs = 30
```

### Étape 4 — Configurer UF2 et HF1 comme deployment clients

```bash
# Sur UF2
ssh uf2
cd /opt/splunkforwarder/bin
./splunk set deploy-poll ds1:8089
./splunk restart
./splunk show deploy-poll                       # Vérifier : ds1:8089
./splunk btool deploymentclient list --debug
exit

# Sur HF1
ssh hf1
cd /opt/splunk/bin
./splunk set deploy-poll ds1:8089
./splunk restart
./splunk show deploy-poll
./splunk btool deploymentclient list --debug
exit
```

### Étape 5 — Créer les server classes (via Splunk Web sur ds1)

**Settings → Forwarder Management → Groups/Server Classes → + New server class**

| Server Class | App assignée | Client |
|-------------|-------------|--------|
| `eng_hf` | hf_base | hf1 |
| `eng_uf` | uf_base | uf2 |

Pour chaque server class : activer **Restart agent**.

### Vérification après déploiement

```bash
# Sur HF1
ssh hf1
cd /opt/splunk/etc/apps
ls -t                                           # hf_base doit apparaître
cat /opt/splunk/etc/apps/hf_base/local/outputs.conf
cd /opt/splunk/bin
./splunk list forward-server                    # idx1 et idx2 actifs
exit

# Sur UF2
ssh uf2
cd /opt/splunkforwarder/etc/apps
ls -t                                           # uf_base doit apparaître
cat /opt/splunkforwarder/etc/apps/uf_base/local/outputs.conf
exit
```

```spl
# Vérification finale sur sh1
index=_internal sourcetype=splunkd host=* | stats count by host
# Doit montrer : hf1, idx1, idx2, sh1, uf1, uf2
```

---

## Lab 13 — Monitor Inputs

### Objectifs
- Déployer un directory monitor via Add Data → Forward vers UF2
- Modifier manuellement inputs.conf et redéployer
- Réinitialiser les checkpoints fishbucket pour ré-indexer

### Déploiement initial via Add Data → Forward

```
Settings → Add Data → Forward
Server Class : eng_webservers | host : uf2
Source : /opt/log | Includelist : www | Excludelist : secure
Index : test
```

### Modifier inputs.conf manuellement sur ds1

```bash
nano /opt/splunk/etc/deployment-apps/_server_app_eng_webservers/local/inputs.conf
```

```ini
[monitor:///opt/log]
blacklist = secure
disabled = false
index = sales         ← changer de test à sales
whitelist = www
host = www            ← ajouter
```

```bash
# Recharger le deployment server
/opt/splunk/bin/splunk reload deploy-server
```

### Réinitialiser le fishbucket sur UF2 (forcer la ré-indexation)

```bash
ssh uf2
cd /opt/splunkforwarder/bin
./splunk stop

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/www1/access.log --reset

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/www2/access.log --reset

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/www3/access.log --reset

./splunk start
exit
```

> ⚠️ Chaque commande `btprobe --reset` doit être sur **une seule ligne**.

### Vérification

```spl
# Sur sh1
index=sales host=www
# Doit montrer : host=www, 3 sources (/opt/log/www1,2,3), sourcetype=access_combined_wcookie
```

---

## Lab 14 — Network Inputs

### Objectifs
- Déployer un input TCP port 9000 vers UF2
- Modifier connection_host et l'index

### Déploiement initial

```
Settings → Add Data → Forward
Server Class : dcrusher_tcp | host : uf2
Source : TCP port 9000 | Source name override : dcrusher9000
Sourcetype : dcrusher | Index : test
```

### Modifier inputs.conf

```bash
nano /opt/splunk/etc/deployment-apps/_server_app_dcrusher_tcp/local/inputs.conf
```

```ini
[tcp://:9000]
connection_host = none           ← changer
host = dcrusher_devserver        ← ajouter
index = itops                    ← changer de test à itops
source = dcrusher9000
sourcetype = dcrusher
```

```bash
/opt/splunk/bin/splunk reload deploy-server
```

---

## Lab 15 — Scripted Inputs

### Objectifs
- Déployer un scripted input vmstat vers UF2
- Vérifier et désactiver l'app

### Préparer le script

```bash
cp /opt/scripts/myvmstat.sh /opt/splunk/bin/scripts
more /opt/splunk/bin/scripts/myvmstat.sh
# Contient : #!/bin/sh \n /usr/bin/vmstat
```

### Déploiement

```
Settings → Add Data → Forward
Server Class : devserver_vmstat | host : uf2
Source : Scripts → $SPLUNK_HOME/bin/scripts/myvmstat.sh | Interval : 30
Sourcetype : vmstat | Index : itops
```

### Inputs.conf généré

```ini
[script://$SPLUNK_HOME/bin/scripts/myvmstat.sh]
disabled = false
interval = 30.0
sourcetype = vmstat
index = itops
```

### Désactiver l'app

```
Settings → Forwarder Management → Configurations → _server_app_devserver_vmstat → Uninstall
```

---

## Lab 16 — Agentless Inputs (HEC)

### Objectifs
- Activer HEC sur ds1
- Créer un token iot_sensors
- Envoyer des événements via curl

### Configuration HEC

```
Settings → Data Inputs → HTTP Event Collector → Global Settings
- All Tokens : Enabled
- Default Source Type : json_no_timestamp
- Default Index : test
- Enable SSL : off
- HTTP Port : 8088
```

### Inputs.conf résultant

```bash
# App globale HEC
more /opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf
# [http]  disabled = 0  enableSSL = 0  index = test  sourcetype = json_no_timestamp

# Token spécifique
more /opt/splunk/etc/apps/search/local/inputs.conf
# [http://iot_sensors]  disabled = 0  index = test  indexes = itops,test  token = <token>
```

### Envoyer des événements via curl (depuis UF1)

```bash
ssh uf1
export H_TOKEN=<token_généré>
export H_SERVER=ds1:8088

# Vérifier les variables
echo $H_TOKEN
echo $H_SERVER

# Envoyer des événements basiques
/opt/scripts/hec1.sh

# Envoyer des événements avec metadata override
/opt/scripts/hec2.sh
exit
```

### Commandes curl brutes

```bash
# Envoi basique
curl "http://${H_SERVER}/services/collector" \
  -H "Authorization: Splunk ${H_TOKEN}" \
  -d '{"event": "Hello World 1"}'

# Envoi avec metadata overridée
curl "http://${H_SERVER}:8088/services/collector" \
  -H "Authorization: Splunk ${H_TOKEN}" \
  -d '{"index":"itops", "host":"uf1", "sourcetype":"custom", "source":"app.log",
       "event":{"code":"200", "status":"OK", "message":"test"}}'
```

### Vérification btool HEC

```bash
/opt/splunk/bin/./splunk btool inputs list http://iot_sensors --debug
```

---

## Lab 17 — Fine-tune Inputs

### Objectifs
- Déployer un directory monitor avec auto-sourcetype
- Surcharger le sourcetype d'un fichier spécifique via props.conf

### Créer props.conf dans deployment-apps

```bash
nano /opt/splunk/etc/deployment-apps/_server_app_devserver_vmail/local/props.conf
```

```ini
[source::/opt/log/vmail/iis_vmail3.log]
sourcetype = acme_voip
```

### Modifier l'index dans inputs.conf

```bash
nano /opt/splunk/etc/deployment-apps/_server_app_devserver_vmail/local/inputs.conf
```

```ini
[monitor:///opt/log/vmail]
disabled = false
index = itops     ← changer de test à itops
```

```bash
/opt/splunk/bin/splunk reload deploy-server
```

### Réinitialiser les checkpoints (pour 2 fichiers)

```bash
ssh uf2
cd /opt/splunkforwarder/bin
./splunk stop

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/vmail/iis_vmail2.log --reset

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/vmail/iis_vmail3.log --reset

./splunk start
exit
```

### Vérification

```spl
index=itops source=*vmail* host=uf2 | stats count by source, sourcetype
# iis_vmail2.log → sourcetype=iis_vmail (auto)
# iis_vmail3.log → sourcetype=acme_voip (surchargé)
```

---

## Lab 18 — Create a New Source Type

### Objectifs
- Créer deux sourcetypes custom via Data Preview
- Déployer props.conf vers HF1
- Déployer les inputs vers UF2

### Sourcetypes créés

| Nom | Fichier source | Particularité |
|-----|---------------|---------------|
| `dc_mem_crash` | crash-*.log | Multi-lignes = 1 événement, `MAX_TIMESTAMP_LOOKAHEAD = 30` |
| `dcrusher_attacks` | dreamcrusher.xml | XML, `LINE_BREAKER = ([\r\n]+)\s*<Interceptor>`, timestamp `<ActionDate>` |

### Props.conf générés

```ini
[dc_mem_crash]
DATETIME_CONFIG =
LINE_BREAKER = ([\r\n]+)
MAX_TIMESTAMP_LOOKAHEAD = 30
NO_BINARY_CHECK = true
category = Application
description = Dream Crusher server memory dump
pulldown_type = true

[dcrusher_attacks]
BREAK_ONLY_BEFORE_DATE =
DATETIME_CONFIG =
LINE_BREAKER = ([\r\n]+)\s*<Interceptor>
NO_BINARY_CHECK = true
SHOULD_LINEMERGE = false
TIME_FORMAT = %Y-%m-%d
TIME_PREFIX = <ActionDate>
TZ = America/Los_Angeles
category = Application
description = Dream Crusher user interactions
pulldown_type = true
```

### Copier props.conf vers hf_base et déployer

```bash
# Sur ds1
cp /opt/splunk/etc/apps/search/local/props.conf \
   /opt/splunk/etc/deployment-apps/hf_base/local/

/opt/splunk/bin/splunk reload deploy-server

# Vérifier sur HF1
ssh hf1
cat /opt/splunk/etc/apps/hf_base/local/props.conf
exit
```

### Réinitialiser checkpoint pour ré-indexation

```bash
ssh uf2
cd /opt/splunkforwarder/bin
./splunk stop

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/crashlog/dreamcrusher.xml --reset

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/crashlog/crash-<xxxx-xx-xx-xx_xx_xx>.log --reset

./splunk start
exit
```

---

## Lab 19 — Manipulating Input Data

### Objectifs (Ex. 1 — Ingest Actions)
- Masquer `AcctCode` dans sales_entries via Ingest Actions
- Observer les props.conf/transforms.conf générés

### Résultat Ingest Actions — props.conf

```ini
[sales_entries]
RULESET-Mask_Sales_Entries = _rule:Mask_Sales_Entries:mask::xxxxxxxx
RULESET_DESC-Mask_Sales_Entries = Mask account codes in the sales_entries.log
```

### Résultat Ingest Actions — transforms.conf

```ini
[_rule:Mask_Sales_Entries:mask::xxxxxxxx]
INGEST_EVAL = _raw:=replace(_raw, "(AcctCode=\\d{4})-\\d{4}", "\\1-XXXX")
```

### Objectifs (Ex. 2 — Méthode classique)
- Masquer `AcctID` dans vendor_sales via transforms.conf + props.conf manuels

### transforms.conf (masquage AcctID)

```ini
[mask-acctid]
REGEX = (.*AcctID=\d{10})\d{6}
DEST_KEY = _raw
FORMAT = $1XXXXXX
```

### props.conf (invoquer le masquage)

```ini
[vendor_sales]
TRANSFORMS-acctmasking = mask-acctid
```

### Copier et déployer vers HF1

```bash
cp /opt/splunk/etc/apps/search/local/props.conf \
   /opt/splunk/etc/deployment-apps/hf_base/local/

cp /opt/splunk/etc/apps/search/local/transforms.conf \
   /opt/splunk/etc/deployment-apps/hf_base/local/

/opt/splunk/bin/splunk reload deploy-server

# Vérifier sur HF1
ssh hf1
cat /opt/splunk/etc/apps/hf_base/local/props.conf
cat /opt/splunk/etc/apps/hf_base/local/transforms.conf
exit
```

---

## Lab 20 — Routing Input Data

### Objectifs (Ex. 1 — Ingest Actions)
- Dropper les événements London de badge_access via Ingest Actions

### Résultat Ingest Actions — transforms.conf

```ini
[_rule:Badge_Access_US:filter:regex:xxxxxxxx]
INGEST_EVAL = queue=if(match(_raw, "London"), "nullQueue", queue)
STOP_PROCESSING_IF = queue == "nullQueue"
```

### Objectifs (Ex. 2 — Méthode classique)
- Router/filtrer sysmonitor.log avec transforms.conf + props.conf

### transforms.conf (filtrage et routing)

```ini
[eventsDrop]
REGEX = Information
DEST_KEY = queue
FORMAT = nullQueue

[eventsRoute]
REGEX = (Error|Warning)
DEST_KEY = _MetaData:Index
FORMAT = itops
```

### props.conf (invoquer le routing)

```ini
[win_audits]
INDEXED_EXTRACTIONS = csv
TRANSFORMS-information = eventsDrop
TRANSFORMS-itops = eventsRoute
```

### Vérification

```spl
# Sur ds1 (résultats avec champs parsés)
index=* source=*sysmonitor.log host=splunk01
| stats count by Type, index, host | sort index, Type

# Sur sh1 (rex nécessaire car indexed extraction non configurée sur indexers)
index=* source=*sysmonitor.log host=uf2
| rex "\,(?<Type>Error|FailureAudit|SuccessAudit|Warning|Information)\,"
| stats count by Type, index, host | sort index, Type
```

---

## Lab 21 — Support Knowledge Objects

### Objectifs
- Créer un report (knowledge object)
- Rechercher et réassigner des KOs orphelins

### Créer un report

```spl
index=test sourcetype=access* | stats count by productId
# Save As → Report → "CountByProductId"
```

### Rechercher les KOs orphelins

```
Settings → All configurations → Reassign Knowledge Objects
→ Cliquer "Orphaned"
→ Filtrer par owner si nécessaire
→ Cliquer "Reassign" → choisir emaxwell
```

---

---

## Référence complète des commandes CLI

### Service Splunk

```bash
./splunk start
./splunk stop
./splunk restart
./splunk status
./splunk login
./splunk version

# Démarrage sans interaction
./splunk start --accept-license
./splunk start --accept-license --answer-yes --no-prompt --seed-passwd <password>

# Boot-start (sans mot de passe à chaque démarrage)
./splunk stop
sudo ./splunk enable boot-start -systemd-managed 0 -user student
./splunk start
```

### Informations système

```bash
./splunk show web-port                  # 8000
./splunk show splunkd-port             # 8089
./splunk show appserver-ports          # 8065
./splunk show kvstore-port             # 8191
./splunk show servername               # Nom du serveur
./splunk show default-hostname         # Hostname par défaut des inputs

./splunk set servername <name>         # Changer le nom
./splunk set default-hostname <name>   # Changer le hostname par défaut
./splunk set splunkd-port <port>       # Changer le port splunkd
```

### Btool — débogage de configuration

```bash
./splunk btool check                                        # Vérifier les erreurs
./splunk btool server list general                          # Config générale server.conf
./splunk btool server list general --debug                  # Avec source des paramètres
./splunk btool server list config_change_tracker

./splunk btool inputs list                                  # Tous les inputs
./splunk btool inputs list splunktcp --debug                # Port de réception S2S
./splunk btool inputs list splunktcp://9901 --debug
./splunk btool inputs list monitor:///var/log --debug
./splunk btool inputs list http://iot_sensors --debug

./splunk btool outputs list tcpout:default-autolb-group --debug  # Config forwarding
./splunk btool props list --debug                           # Tous les props
./splunk btool props list source::/opt/log/vmail/iis_vmail --debug
./splunk btool deploymentclient list --debug                # Config deployment client

./splunk show config server                                 # Config server en mémoire
./splunk show config server | grep serverName
./splunk show config inputs                                 # Config inputs en mémoire
```

### Forwarding et réception

```bash
# Sur l'indexer — configurer le port de réception
./splunk enable listen 9997
./splunk enable listen 9901                    # Port pour forwarder intermédiaire
./splunk display listen                        # Afficher le port de réception actuel

# Sur le forwarder — configurer les destinations
./splunk add forward-server ds1:9997
./splunk add forward-server idx1:9997
./splunk add forward-server idx2:9997
./splunk add forward-server hf1:9901

./splunk remove forward-server ds1:9997
./splunk remove forward-server idx1:9997

./splunk list forward-server                   # Lister les destinations (active/inactive)
```

### Monitor inputs (sur forwarder)

```bash
./splunk list monitor                          # Lister les fichiers surveillés

./splunk add monitor /opt/log/www1/access.log
./splunk add monitor /var/log/ -index web -sourcetype access_combined -host webserver01
./splunk add monitor /opt/log/www1/access.log -sourcetype access_combined_wcookie -index test
```

### Deployment Server et clients

```bash
# Sur le DS
./splunk reload deploy-server                  # Recharger les apps et checksums
./splunk list deploy-clients                   # Lister les clients connectés

# Sur le client (forwarder)
./splunk set deploy-poll ds1:8089              # Configurer le DS
./splunk show deploy-poll                      # Vérifier : "Deployment Server URI is set to ds1:8089"
./splunk disable deploy-client                 # Désactiver le polling
./splunk btool deploymentclient list --debug   # Vérifier la config
```

### Gestion des index

```bash
./splunk add index <index_name>
./splunk add index <name> -app <app_name>

./splunk clean eventdata -index <index_name>   # ⚠️ IRRÉVERSIBLE
./splunk clean eventdata                        # Tous les index ⚠️
```

### Fishbucket — réinitialisation de checkpoints

```bash
# Réinitialiser un fichier spécifique (forcer la ré-indexation)
./splunk stop

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/www1/access.log --reset

./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db \
    --file /opt/log/crashlog/dreamcrusher.xml --reset

./splunk start

# Réinitialiser tout le fishbucket (⚠️ ré-indexe TOUT)
cd /opt/splunkforwarder/var/lib/splunk/
rm -r fishbucket
./splunk restart

# Inspecter le fishbucket
./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db
```

### Users

```bash
./splunk edit user <user> -locked-out false -auth admin:<password>
```

### Diagnostics

```bash
./splunk diag                                              # Générer un bundle diag

# Tester un script dans le contexte Splunk
./splunk cmd /opt/splunk/etc/apps/my_app/bin/myscript.sh

# Vérifier les erreurs/warnings dans splunkd.log
tail /opt/splunkforwarder/var/log/splunk/splunkd.log | grep 'ERROR\|WARN'
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep -E 'TcpInputConfig.*9000'
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep 'ExecProcessor'
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep 'TailingProcessor'
```

---

## Fichiers de configuration produits dans les labs

### inputs.conf (ds1 / search app)

```ini
[monitor:///opt/log/www2/access.log]
disabled = false
index = securityops
sourcetype = access_combined_wcookie

[monitor:///opt/log/www3/access.log]
disabled = false
index = test
sourcetype = access_combined_wcookie

[splunktcp://9997]
connection_host = ip

[http]
disabled = 0
enableSSL = 0
index = test
sourcetype = json_no_timestamp

[http://iot_sensors]
disabled = 0
index = test
indexes = itops,test
token = <generated_token>
```

### inputs.conf (UF1 / search app)

```ini
[monitor:///opt/log/www1/access.log]
disabled = false
index = test
sourcetype = access_combined_wcookie
```

### inputs.conf (HF1 / hf_base app — déployé par DS)

```ini
[splunktcp://9901]
connection_host = ip
```

### outputs.conf (UF1 → indexers)

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
disabled = false
server = idx1:9997,idx2:9997
```

### outputs.conf (UF2 → HF1 / uf_base app — déployé par DS)

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout-server://hf1:9901]

[tcpout:default-autolb-group]
disabled = false
server = hf1:9901
```

### outputs.conf (HF1 → indexers / hf_base app — déployé par DS)

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout-server://idx1:9997]
[tcpout-server://idx2:9997]

[tcpout:default-autolb-group]
disabled = false
server = idx1:9997,idx2:9997
```

### indexes.conf (search app / ds1)

```ini
[securityops]
coldPath = $SPLUNK_DB/securityops/colddb
homePath = $SPLUNK_DB/securityops/db
maxDataSize = auto_high_volume
maxTotalDataSizeMB = 512000
thawedPath = $SPLUNK_DB/securityops/thaweddb
maxHotSpanSecs = 86400
frozenTimePeriodInSecs = 7776000

[volume:one]
path = /home/student/one/
maxVolumeDataSizeMB = 40000

[volume:two]
path = /home/student/two/
maxVolumeDataSizeMB = 80000

[itops]
coldPath = volume:two/itops/colddb
homePath = volume:one/itops/db
maxTotalDataSizeMB = 512000
thawedPath = $SPLUNK_DB/itops/thaweddb
homePath.maxDataSizeMB = 30000
coldPath.maxDataSizeMB = 60000
```

### props.conf (hf_base — déployé sur HF1)

```ini
[dc_mem_crash]
DATETIME_CONFIG =
LINE_BREAKER = ([\r\n]+)
MAX_TIMESTAMP_LOOKAHEAD = 30
NO_BINARY_CHECK = true
category = Application
description = Dream Crusher server memory dump
pulldown_type = true

[dcrusher_attacks]
BREAK_ONLY_BEFORE_DATE =
DATETIME_CONFIG =
LINE_BREAKER = ([\r\n]+)\s*<Interceptor>
NO_BINARY_CHECK = true
SHOULD_LINEMERGE = false
TIME_FORMAT = %Y-%m-%d
TIME_PREFIX = <ActionDate>
TZ = America/Los_Angeles
category = Application
description = Dream Crusher user interactions
pulldown_type = true

[vendor_sales]
TRANSFORMS-acctmasking = mask-acctid

[win_audits]
INDEXED_EXTRACTIONS = csv
TRANSFORMS-information = eventsDrop
TRANSFORMS-itops = eventsRoute
```

### transforms.conf (hf_base — déployé sur HF1)

```ini
[mask-acctid]
REGEX = (.*AcctID=\d{10})\d{6}
DEST_KEY = _raw
FORMAT = $1XXXXXX

[eventsDrop]
REGEX = Information
DEST_KEY = queue
FORMAT = nullQueue

[eventsRoute]
REGEX = (Error|Warning)
DEST_KEY = _MetaData:Index
FORMAT = itops
```

---

## Commandes de troubleshooting

### Vérifier le statut général

```bash
./splunk status
./splunk btool check
tail /opt/splunk/var/log/splunk/splunkd.log | grep 'ERROR\|WARN'
```

### Problèmes de forwarding

```bash
# Vérifier les destinations actives
./splunk list forward-server

# Vérifier la config outputs.conf
./splunk btool outputs list tcpout:default-autolb-group --debug

# Vérifier le port de réception
./splunk display listen
./splunk btool inputs list splunktcp --debug
```

### Problèmes de deployment server

```bash
# Sur le client
./splunk show deploy-poll
./splunk set deploy-poll ds1:8089
./splunk restart

# Sur le DS
./splunk reload deploy-server
./splunk list deploy-clients

# Vérifier les apps déployées
ls /opt/splunk/etc/deployment-apps/
ls /opt/splunk/etc/apps/          # Sur le client après déploiement
```

### Problèmes d'indexation / inputs

```bash
# Vérifier la config d'un monitor
./splunk btool inputs list monitor:///opt/log --debug

# Vérifier les checkpoints fishbucket
./splunk cmd btprobe -d /opt/splunkforwarder/var/lib/splunk/fishbucket/splunk_private_db

# Voir les erreurs de monitoring
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep 'TailingProcessor'

# Vérifier les erreurs TCP
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep -E 'TcpInputConfig.*9000'
cat /opt/splunkforwarder/var/log/splunk/splunkd.log | grep -E 'TcpInputProc.*9000'
```

### Problèmes HEC

```bash
# Vérifier la config HEC
more /opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf
./splunk btool inputs list http://iot_sensors --debug

# Test curl direct
curl http://ds1:8088/services/collector \
  -H "Authorization: Splunk <token>" \
  -d '{"event": "test"}'
```

### Problèmes de parsing / sourcetype

```bash
# Vérifier props.conf fusionné
./splunk btool props list source::/opt/log/vmail/iis_vmail --debug

# Forcer la ré-indexation après correction
./splunk stop
./splunk cmd btprobe -d /path/fishbucket/splunk_private_db --file /path/to/file --reset
./splunk start
```

### Recherches SPL de troubleshooting

```spl
-- Métriques TCP pour un forwarder
index=_internal host=uf2 component=Metrics name=tcpin_queue

-- Erreurs ExecProcessor (scripts)
index=_internal sourcetype=splunkd component=ExecProcessor host=uf2

-- Vérifier tous les hôtes qui indexent
index=_internal sourcetype=splunkd host=* | stats count by host

-- Taux d'indexation par sourcetype
index=_internal component=Metrics group=per_sourcetype_thruput
| stats sum(kbps) by series
```

---
