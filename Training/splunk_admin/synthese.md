# Splunk Enterprise Administrator Bootcamp v9.4 — Synthèse de révision

> Source : Splunk Enterprise Administrator Bootcamp Slides v9.4 (Kevin Stewart, 2025)  
> 21 modules · 486 slides

---

## Sommaire

1. [Module 1 — Deploy Splunk](#module-1--deploy-splunk)
2. [Module 2 — Monitor Splunk](#module-2--monitor-splunk)
3. [Module 3 — License Splunk](#module-3--license-splunk)
4. [Module 4 — Use Configuration Files](#module-4--use-configuration-files)
5. [Module 5 — Use Apps](#module-5--use-apps)
6. [Module 6 — Create Indexes](#module-6--create-indexes)
7. [Module 7 — Manage Indexes](#module-7--manage-indexes)
8. [Module 8 — Manage Users](#module-8--manage-users)
9. [Module 9 — Get Data Into Splunk](#module-9--get-data-into-splunk)
10. [Module 10 — Configure Forwarders](#module-10--configure-forwarders)
11. [Module 11 — Customize Forwarders](#module-11--customize-forwarders)
12. [Module 12 — Manage Forwarders (Deployment Server)](#module-12--manage-forwarders)
13. [Module 13 — Monitor Inputs](#module-13--monitor-inputs)
14. [Module 14 — Network Inputs](#module-14--network-inputs)
15. [Module 15 — Scripted Inputs](#module-15--scripted-inputs)
16. [Module 16 — Agentless Inputs (HEC)](#module-16--agentless-inputs-hec)
17. [Module 17 — Fine-tune Inputs](#module-17--fine-tune-inputs)
18. [Module 18 — Parsing Phase & Data Preview](#module-18--parsing-phase--data-preview)
19. [Module 19 — Manipulate Input Data](#module-19--manipulate-input-data)
20. [Module 20 — Route Input Data](#module-20--route-input-data)
21. [Module 21 — Support Knowledge Objects](#module-21--support-knowledge-objects)
22. [Commandes CLI — référence complète](#commandes-cli--référence-complète)
23. [Fichiers de configuration — référence](#fichiers-de-configuration--référence)
24. [Ports réseau par défaut](#ports-réseau-par-défaut)
25. [Pièges et best practices](#pièges-et-best-practices)

---

## Module 1 — Deploy Splunk

### Types de déploiements

| Type | Description | Usage |
|------|-------------|-------|
| **Standalone** | Tout sur un seul serveur | Tests, POC, apprentissage |
| **Single server with inputs** | Serveur + forwarders | Petits environnements |
| **Distributed non-cluster** | Forwarders → Indexers → Search Heads séparés | Production standard |
| **Clustered** | Index clusters + Search Head clusters | Haute disponibilité, gros volumes |

### Composants Splunk

| Composant | Rôle | Processus |
|-----------|------|-----------|
| **Search Head** | Interface web, distribution des recherches | `splunkd` + `splunkweb` |
| **Indexer** | Parsing, indexation, stockage des buckets | `splunkd` |
| **Universal Forwarder (UF)** | Collecte légère, transfert vers indexer | `splunkd` |
| **Heavy Forwarder (HF)** | Collecte + parsing local possible | `splunkd` |
| **Deployment Server** | Gestion centralisée des configurations | `splunkd` |
| **License Manager** | Gère les licences pour les peers | `splunkd` |
| **Monitoring Console (MC)** | Surveillance de l'infrastructure Splunk | App Splunk dédiée |

### 4 phases du pipeline de données

```
Input → Parsing → Indexing → Searching
```

1. **Input** : Ouverture et lecture des sources de données (stream de bytes)
2. **Parsing** : Découpage en événements, extraction du timestamp (sur indexer ou HF)
3. **Indexing** : License meter, compression, écriture sur disque
4. **Searching** : Interrogation des index via SPL

> ⚠️ Une fois la donnée indexée sur disque, elle **ne peut plus être modifiée**.

### Installation Splunk Enterprise sur Linux

```bash
# Démarrage initial (crée le compte admin)
./splunk start --accept-license --answer-yes --no-prompt --seed-passwd <password>

# Activer le démarrage automatique au boot
./splunk enable boot-start -user splunk

# Vérifier le statut
./splunk status
```

### Best Practices — Installation

- Utiliser un **compte non-root dédié** pour faire tourner Splunk (`splunk` user)
- Synchroniser les horloges avec **NTP** sur tous les serveurs (critique pour les timestamps)
- Configurer les **ulimits** système (file descriptors, process) sur les indexers
- Désactiver le port **8089 (splunkd management)** sur les Universal Forwarders en production si non nécessaire

---

## Module 2 — Monitor Splunk

### Options de monitoring

| Outil | Description |
|-------|-------------|
| **Health Report** | Tableau de bord intégré, alertes sur l'état de santé de l'instance |
| **Monitoring Console (MC)** | App admin dédiée, dashboards prédéfinis pour search, indexing, forwarders, ressources |
| **Splunk Diag** | Outil de diagnostic qui génère un bundle `.tar.gz` (logs, configs, métadonnées) |
| **Rapid Diag** | Diag interactif via l'interface MC |
| **Splunk Observability** | Produit séparé pour l'observabilité avancée |

### Health Report Manager

- Accessible via **Settings → Health Report Manager**
- Permet de modifier les seuils d'alerte et de "snoozer" des alertes temporairement
- Surveille : search scheduler, license usage, index pipeline, forwarder connectivity...

### Monitoring Console

- App admin disponible sur Splunk Enterprise et Cloud
- En mode **standalone** : surveille le serveur local
- En mode **distributed** : surveille tous les composants du cluster
- Contenus principaux : Search · Indexing · Resources · Forwarders · Instances

### Splunk Diag

```bash
# Génère un bundle de diagnostique
./splunk diag

# Diag ciblé (exemples de composants)
./splunk diag --collect=etc,log,index_files
```

> Le diag collecte : configs, logs, métadonnées d'index. **Pas de données client ni de données indexées.** Le fichier tar.gz peut lui-même être ingéré dans Splunk pour analyse.

---

## Module 3 — License Splunk

### Types de licences

| Type | Limite | Fonctionnalités |
|------|--------|-----------------|
| **Enterprise Trial** | 500 MB/jour · 60 jours | Identique à Enterprise |
| **Enterprise** | Volume acheté (GB/jour) | Toutes fonctionnalités, no-enforcement si ≥ 100 GB/jour |
| **Free** | 500 MB/jour | Pas d'alertes, auth, clustering, distributed search |
| **Forwarder** | Illimité pour transfert | Pas d'indexation locale, auth OK |

### Ingest vs Workload Pricing

| Modèle | Base | Monitoring |
|--------|------|-----------|
| **Ingest** | Volume de données indexées (GB/jour) | MC + Settings → Licensing |
| **Workload** | Capacité de calcul (vCPUs / SVCs) | MC (vCPUs) ou Cloud MC |

### Ce qui compte dans le quota journalier

**Compte :** Toutes les données indexées (taille full avant compression)  
**Ne compte pas :**
- Données répliquées (Index Clusters)
- Summary indexes
- Index internes (`_internal`, `_audit`, etc.)
- Métadonnées structurelles (tsidx, metadata)
- Métriques : plafonnées à **150 bytes/événement**

### Violations de licence

- Une violation = dépassement du quota journalier
- **5 violations en 30 jours** → les recherches sont bloquées (sauf licences no-enforcement ≥ 100 GB/jour)
- Configurer une alerte MC sur l'usage des licences

### Architecture License Manager

```
License Manager (LM)
    └── License Peers (indexers) → phone home au LM
```

- Le LM peut être une instance Splunk Enterprise dédiée
- Les peers se connectent au LM via le port **8089**

---

## Module 4 — Use Configuration Files

### Structure des fichiers de configuration

```
SPLUNK_HOME/etc/
├── system/
│   ├── default/     ← valeurs par défaut Splunk (ne jamais modifier)
│   ├── local/       ← overrides système (éviter, préférer une app)
│   └── README/      ← .conf.spec et .conf.example (documentation)
└── apps/
    └── <app_name>/
        ├── default/ ← valeurs par défaut de l'app
        └── local/   ← overrides locaux de l'app (modifié par Splunk Web/CLI)
```

### Format des fichiers .conf

```ini
[default]
attribut = valeur_globale

[stanza_name]
attribut1 = valeur1
attribut2 = valeur2
```

> Les fichiers .conf sont généralement **sensibles à la casse**.

### Layering (priorité des configurations)

Du plus prioritaire au moins prioritaire :

```
1. SPLUNK_HOME/etc/apps/<app>/local/    ← priorité max (modif UI/CLI)
2. SPLUNK_HOME/etc/apps/<app>/default/
3. SPLUNK_HOME/etc/system/local/
4. SPLUNK_HOME/etc/system/default/      ← priorité min (valeurs Splunk)
```

> ⚠️ Les settings **index-time** ne peuvent pas être overridés depuis une app sur un indexer — il faut modifier `system/local`. Préférer créer une app dédiée (ex. `DC_app`) déployée sur les indexers via le Deployment Server.

### Fichiers .conf principaux

| Fichier | Où | Rôle |
|---------|----|------|
| `inputs.conf` | Forwarder, Indexer | Définit les sources de données collectées |
| `outputs.conf` | Forwarder | Définit où transférer les données |
| `props.conf` | Indexer, Search Head | Parsing, extractions, lookup, source type settings |
| `transforms.conf` | Indexer, Search Head | Transformations, extractions regex, routing |
| `indexes.conf` | Indexer | Définit et configure les index |
| `serverclass.conf` | Deployment Server | Définit les server classes |
| `fields.conf` | Search Head | Champs indexés additionnels |

### Outils de validation de configuration

```bash
# Vérifier la configuration fusionnée d'un fichier
./splunk btool inputs list --debug
./splunk btool inputs list monitor:///var/log --debug

# Vérifier les erreurs de configuration
./splunk btool check

# Voir la configuration effective (fusion de toutes les couches)
./splunk show config inputs

# Recharger sans restart (selon le fichier)
./splunk reload deploy-server
```

> **btool** = "Bundles Tool" — lit la configuration telle que Splunk la voit après layering. L'option `--debug` affiche le fichier source de chaque attribut.

### Désactiver un attribut hérité

Pour "annuler" un attribut hérité d'une couche inférieure, l'assigner à une valeur vide :

```ini
[syslog]
TRANSFORMS =      ← désactive l'attribut TRANSFORMS hérité
```

---

## Module 5 — Use Apps

### Apps vs Add-ons

| | App | Add-on |
|--|-----|--------|
| **Interface Splunk Web** | Oui (navigation propre, URL dédiée) | Non |
| **Namespace** | Oui | Oui |
| **Redistribuable** | Oui | Oui |
| **Usage typique** | Interface utilisateur + données | Collecte de données, extractions |

### Structure d'une app

```
SPLUNK_HOME/etc/apps/<app_name>/
├── app.conf           ← métadonnées de l'app
├── default/
│   ├── props.conf
│   ├── transforms.conf
│   ├── inputs.conf
│   └── ...
├── local/             ← overrides locaux
├── metadata/
│   └── default.meta   ← permissions des objets
├── bin/               ← scripts
└── lookups/           ← fichiers CSV lookup
```

### Apps par défaut

| App | ID | Description |
|-----|-----|-------------|
| **Home** | `launcher` | Accès aux apps, aide, tutoriels |
| **Search and Reporting** | `search` | Environnement de recherche complet |
| **Upgrade Readiness App** | — | Vérifie la compatibilité Python 3 des apps |
| **Sample Data** | `sample_app` | Template pour créer une app (caché par défaut) |

### Gestion des apps

```bash
# Lister les apps installées
./splunk display app

# Installer une app depuis un fichier .tgz
./splunk install app /path/to/app.tgz

# Supprimer une app
./splunk remove app <app_name>
```

### Permissions

Les permissions se configurent dans **`metadata/default.meta`** ou via Settings → Apps. Niveaux : Private / App / Global.

---

## Module 6 — Create Indexes

### Rôle des index

- Stockent les données sous forme d'événements
- Permettent le **contrôle d'accès** (par rôle → accès restreint à certains index)
- Permettent des **politiques de rétention différenciées** (un index = une politique)
- Stockés dans `SPLUNK_DB` = `SPLUNK_HOME/var/lib/splunk`

### Types d'index

| Type | Description |
|------|-------------|
| **Events index** | Logs non structurés, données textuelles (type par défaut) |
| **Metrics index** | Métriques système/applicatives optimisées pour ingestion et analyse rapide |

### Index préconfigurés importants

| Index | Rôle |
|-------|------|
| `main` | Index par défaut pour les données |
| `_internal` | Logs et métriques internes Splunk |
| `_audit` | Trails d'audit Splunk |
| `_introspection` | Performances et ressources système (alimente le MC) |
| `_thefishbucket` | Checkpoints du file monitoring (ne pas toucher) |
| `summary` | Index pour le summary indexing |

### Créer un index

**Via Splunk Web :** Settings → Indexes → New Index

**Via CLI :**
```bash
./splunk add index <index_name> [-app <app_context>]
```

**Via `indexes.conf` :**
```ini
[itops]
homePath   = /mnt/ssd/splunk/itops/db
coldPath   = /mnt/san/splunk/itops/colddb
thawedPath = $SPLUNK_DB/itops/thaweddb
maxDataSize = auto_high_volume     ← taille max bucket : auto=750MB, auto_high_volume=10GB
maxTotalDataSizeMB = 307200        ← taille max totale de l'index = 300 GB
```

> ⚠️ Les chemins `homePath`, `coldPath`, `thawedPath` doivent être spécifiés même si on utilise les valeurs par défaut.

---

## Module 7 — Manage Indexes

### Cycle de vie des buckets

```
hot → warm → cold → frozen (archivé ou supprimé)
```

| Bucket | État | Stockage recommandé |
|--------|------|---------------------|
| **hot** | Actif, en écriture | SSD (disque rapide) |
| **warm** | Récent, en lecture seule | SSD ou disque rapide |
| **cold** | Ancien, en lecture seule | Disque lent (SAN, NAS) |
| **frozen** | Archivé ou supprimé | Archive externe |
| **thawed** | Restauré depuis frozen | Disque local temporaire |

### Chemins des données

```
indexes.conf (définition)  → SPLUNK_HOME/etc/.../
Données (buckets)          → SPLUNK_HOME/var/lib/splunk/<index>/db/       (hot/warm)
                           → SPLUNK_HOME/var/lib/splunk/<index>/colddb/    (cold)
                           → SPLUNK_HOME/var/lib/splunk/<index>/thaweddb/  (thawed)
```

### Paramètres de rétention clés dans `indexes.conf`

```ini
[itops]
frozenTimePeriodInSecs = 31536000   ← rétention max en secondes (ici 1 an)
maxTotalDataSizeMB     = 500000     ← taille max totale (volume-based)
homePath.maxDataSizeMB = 100000     ← taille max du répertoire hot/warm
coldPath.maxDataSizeMB = 400000     ← taille max du répertoire cold
maxHotBuckets          = 3          ← nombre de buckets hot simultanés
maxWarmDBCount         = 300        ← nombre de buckets warm max
maxHotSpanSecs         = 86400      ← durée max d'un bucket hot (1 jour)
```

### Le Fishbucket

- Index spécial `_thefishbucket` qui stocke les **checkpoints** du file monitoring
- Permet à Splunk de savoir où il en est dans la lecture des fichiers
- À réinitialiser lors de tests si on veut ré-indexer un fichier déjà lu

```bash
# Réinitialiser le fishbucket (DANGEREUX en production)
./splunk clean eventdata -index _thefishbucket

# Inspecter le fishbucket
./splunk cmd btprobe -d $SPLUNK_DB/fishbucket/splunk_private_db
```

### Supprimer des données d'un index

```bash
# Supprimer toutes les données d'un index (garde la définition)
./splunk clean eventdata -index <index_name>

# Supprimer toutes les données de tous les index
./splunk clean eventdata
```

> ⚠️ `clean eventdata` est **irréversible**. Ne jamais exécuter en production sans validation.

### Restaurer un bucket frozen (thaw)

1. Copier le bucket dans le répertoire `thaweddb` de l'index
2. Exécuter `./splunk rebuild <chemin_du_bucket>` si nécessaire
3. Redémarrer Splunk — le bucket devient searchable

---

## Module 8 — Manage Users

### Authentification

| Méthode | Description | Priorité |
|---------|-------------|---------|
| **SAML** | SSO (Okta, ADFS...) | 1 (plus haute) |
| **Native Splunk** | Comptes dans `SPLUNK_HOME/etc/passwd` | 2 |
| **LDAP** | Active Directory, OpenLDAP | 3 |

> **Best Practice :** Toujours conserver un compte failsafe dans le fichier `passwd` natif Splunk avec un mot de passe très fort, même si SAML/LDAP est utilisé.

### Rôles intégrés

| Rôle | Capacités principales |
|------|----------------------|
| `admin` | Tout faire, créer des rôles custom |
| `power` | Éditer objets partagés, saved searches, alertes, tagger des events |
| `user` | Créer et lancer ses propres searches, créer event types |
| `can_delete` | Supprimer par mot-clé (nécessaire pour `delete` en SPL) |

### Créer un utilisateur

**Via Splunk Web :** Settings → Users → New User

Paramètres clés :
- **Roles** : un utilisateur hérite des capacités de tous ses rôles (union)
- **Default app** : app affichée au login (défaut : Home ou la default app du rôle)
- **Time zone** : par défaut celle du Search Head

```bash
# Débloquer un utilisateur verrouillé
./splunk edit user <locked_user> -locked-out false -auth admin:<password>
```

### Rôles et héritage

- Un rôle peut **hériter** des capacités d'un autre rôle
- Les **index** accessibles sont définis par liste blanche dans le rôle
- La **quota de recherche** (disk quota, search job limit) est configurable par rôle

---

## Module 9 — Get Data Into Splunk

### Types d'inputs supportés

| Type | Description |
|------|-------------|
| **Files & Directories** | Monitor sur fichiers/répertoires locaux ou distants |
| **Network (TCP/UDP)** | Syslog, données réseau sur un port |
| **Scripted inputs** | Sortie de scripts (sh, py, ps1, bat) |
| **HTTP Event Collector** | REST API HTTP/HTTPS avec token |
| **Windows Event Logs** | Logs Windows natifs |
| **Other** | DB Connect, S3, API, message queues... |

### Métadonnées assignées à l'ingestion

| Métadonnée | Description | Défaut si non spécifié |
|------------|-------------|----------------------|
| `host` | Hôte source de l'événement | Nom du serveur Splunk |
| `source` | Chemin du fichier ou URI | Chemin complet du fichier |
| `sourcetype` | Format/type des données | Auto-détecté par Splunk |
| `index` | Index de destination | `main` |

### Test des inputs — procédure recommandée

1. Utiliser un **serveur de test/dev** (même version que la prod)
2. Créer un **index de test** dédié
3. Copier les données de production sur le serveur de test
4. Utiliser **Splunk Web → Add Data** pour tester
5. Vérifier le sourcetype, les timestamps, les métadonnées
6. Supprimer les données de test, réinitialiser le fishbucket si nécessaire
7. Ajuster la configuration et répéter

---

## Module 10 — Configure Forwarders

### Universal Forwarder (UF)

- **Binaire séparé** (pas la version Enterprise complète)
- Licence intégrée, sans limite de données
- Conçu pour tourner sur les serveurs de production
- **Pas d'interface web, pas de recherche, pas d'indexation locale**
- Bande passante limitée à **256 KBps par défaut** (configurable)
- `SPLUNK_HOME` par défaut : `/opt/splunkforwarder` (*nix) ou `C:\Program Files\SplunkUniversalForwarder` (Windows)

### Étapes de configuration UF

```
1. Configurer un port de réception sur chaque indexer (une seule fois)
2. Télécharger et installer le Universal Forwarder
3. Configurer le forwarding (outputs.conf)
4. Ajouter les inputs (inputs.conf)
```

**Étape 1 — Configurer le port de réception sur l'indexer :**
```bash
./splunk enable listen 9997 [-app app_name]
```
Ou dans `inputs.conf` : `[splunktcp://9997]`

**Étape 3 — Configurer le forwarding sur le forwarder :**
```bash
./splunk add forward-server <indexer_host>:9997
```
Ou dans `outputs.conf` :
```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = indexer1:9997, indexer2:9997
```

**Étape 4 — Ajouter un input sur le forwarder :**
```bash
./splunk add monitor /var/log/access.log -index web -sourcetype access_combined
```

### Heavy Forwarder (HF)

- Instance **Splunk Enterprise complète** configurée pour forwarder
- Peut faire du **parsing local** (props.conf, transforms.conf)
- Peut **router** les données vers différents indexers
- Peut agir comme **forwarder intermédiaire**

---

## Module 11 — Customize Forwarders

### Forwarder intermédiaire

Un Heavy Forwarder placé entre les UF et les indexers permet de :
- **Réduire la bande passante** sur certains segments réseau
- **Traverser les DMZ / firewalls** (un seul point d'entrée)
- **Parser les données** avant l'indexation
- **Router** vers différents indexers selon le contenu

```
UF → HF (parsing/routing) → Indexer A
                           → Indexer B
```

### Options avancées de forwarding (outputs.conf)

```ini
[tcpout]
defaultGroup = my-indexers

[tcpout:my-indexers]
server = idx1:9997, idx2:9997, idx3:9997   ← load balancing automatique
compressed = true                           ← compression (réduit réseau, augmente CPU)
useSSL = true                               ← chiffrement TLS
useACK = true                               ← acknowledgement d'indexation
maxQueueSize = 7MB                          ← taille queue forwarder
```

### Load balancing automatique

Par défaut, le forwarder envoie les données vers un indexer à la fois et **bascule automatiquement** sur le suivant en cas de défaillance. La rotation se fait toutes les 30 secondes par défaut.

---

## Module 12 — Manage Forwarders

### Deployment Server (DS)

Outil intégré pour gérer **centralement** les configurations des forwarders et autres instances Splunk.

```
Deployment Server
├── Deployment Apps  → packages de configuration (ex. inputs.conf) dans etc/deployment-apps/
├── Deployment Clients → instances Splunk qui "phone home" au DS
└── Server Classes → groupes de clients + apps à déployer → serverclass.conf
```

- Interface graphique : **Forwarder Management** (Settings → Forwarder Management)
- Port de communication : **8089** (management port)
- Nécessite une licence **Enterprise**
- Recommandé sur un **serveur dédié**

### Configuration côté DS

```ini
# serverclass.conf — associer clients à apps
[serverClass:linux_servers]
whitelist.0 = *linux*
appList = my_linux_inputs_app

[serverClass:windows_servers]
whitelist.0 = *win*
appList = my_windows_inputs_app
```

### Configuration côté client (forwarder)

```bash
# Configurer le deployment client
./splunk set deploy-poll <DS_hostname>:8089

# Vérifier la configuration
./splunk show deploy-poll

# Désactiver le deployment client
./splunk disable deploy-client
```

### Workflow de déploiement

1. Créer/modifier le deployment app dans `etc/deployment-apps/<app>/local/`
2. Définir les server classes dans `serverclass.conf`
3. Recharger le DS : `./splunk reload deploy-server`
4. Les clients "phone home" toutes les 60 secondes et récupèrent les updates

```bash
# Lister les clients connectés
./splunk list deploy-clients

# Forcer le rechargement du DS
./splunk reload deploy-server
```

---

## Module 13 — Monitor Inputs

### File Monitor

```ini
# inputs.conf
[monitor:///var/log/secure]
disabled   = false
sourcetype = linux_secure
index      = security
host       = webserver01

[monitor:///var/log/apache2/]
disabled   = false
sourcetype = access_combined
index      = web
whitelist  = \.log$           ← surveille seulement les .log
blacklist  = \.gz$            ← exclut les fichiers .gz
```

### Wildcards dans les chemins monitor

| Wildcard | Description |
|----------|-------------|
| `...` | Remplace n'importe quel nombre de répertoires |
| `*` | Remplace n'importe quel fichier ou répertoire (un seul niveau) |
| `?` | Remplace un seul caractère |

Exemples :
```ini
[monitor:///var/log/.../access.log]   ← access.log dans n'importe quel sous-dossier
[monitor:///var/log/www*/]            ← tous les dossiers commençant par www
```

### Fishbucket et file monitoring

Splunk maintient un **checkpoint** pour chaque fichier surveillé dans `_thefishbucket`. Cela lui permet de reprendre la lecture là où il s'était arrêté après un redémarrage ou une rotation de log.

- **Rotation de logs** : Splunk détecte automatiquement les rotations et continue à surveiller le nouveau fichier
- **Fichiers compressés** : décompressés automatiquement (.gz)
- **Formats supportés** : texte plat, CSV, XML, JSON, Log4J multi-lignes

### Déployer un monitor input via Deployment Server

```ini
# Dans etc/deployment-apps/my_monitor_app/local/inputs.conf
[monitor:///var/log/apache2/]
sourcetype = access_combined
index      = web
```

Déployer l'app via les server classes du DS.

---

## Module 14 — Network Inputs

### TCP et UDP inputs

```ini
# inputs.conf
[udp://514]
connection_host = dns
sourcetype      = syslog
index           = network

[tcp://9001]
connection_host = none
host            = dnsserver
sourcetype      = dnslog
```

### Attribut `connection_host`

| Valeur | Comportement | Défaut pour |
|--------|-------------|-------------|
| `dns` | Host = reverse DNS lookup de l'IP source | TCP |
| `ip` | Host = adresse IP de la source | UDP |
| `none` | Host = valeur explicite de l'attribut `host` | — |

> ⚠️ `udp` est **sans connexion** — Splunk ne peut pas garantir la réception. Pour les données critiques, préférer **TCP** ou **HEC avec ACK**.

---

## Module 15 — Scripted Inputs

### Principe

Splunk exécute un script à intervalle régulier et indexe sa sortie stdout.

```ini
# inputs.conf
[script://./bin/myvmstat.sh]
disabled   = false
interval   = 60.0      ← en secondes, ou cron syntax (ex: "*/5 * * * *")
source     = vmstat
sourcetype = myvmstat
index      = os_metrics
```

### Contrainte de sécurité importante

> ⚠️ Splunk n'exécute des scripts **que** depuis ces répertoires :
> - `SPLUNK_HOME/etc/apps/<app_name>/bin/`
> - `SPLUNK_HOME/bin/scripts/`
> - `SPLUNK_HOME/etc/system/bin/`

### Tester un script manuellement

```bash
./splunk cmd SPLUNK_HOME/etc/apps/my_app/bin/myscript.sh
```

### Formats de scripts supportés

`.sh` · `.bat` · `.ps1` · `.py`

---

## Module 16 — Agentless Inputs (HEC)

### HTTP Event Collector (HEC)

Permet d'envoyer des données **sans forwarder**, via HTTP/HTTPS avec un token.

Cas d'usage : applications web, scripts d'automatisation, apps mobiles, environnements legacy.

### Configuration HEC

**Étape 1 — Activer HEC :**
Settings → Data Inputs → HTTP Event Collector → Global Settings → **Enabled**

**Étape 2 — Créer un token :**
New Token → Nommer → Définir sourcetype et index par défaut → Copier le token généré

**Étape 3 — Envoyer des données :**
```bash
curl "https://<splunk_server>:8088/services/collector" \
  -H "Authorization: Splunk <token>" \
  -d '{
    "host": "webserver01",
    "sourcetype": "my_app_logs",
    "source": "app.log",
    "event": {"message": "User login failed", "user": "jdoe", "code": 401}
  }'
```

### Port HEC par défaut

- **8088** (HTTP/HTTPS)

### Options avancées HEC

| Option | Description |
|--------|-------------|
| **ACK** | Le client reçoit une confirmation d'indexation |
| **Raw endpoint** | Envoie des données brutes (pas JSON) sur `/services/collector/raw` |
| **Channel** | Identifiant de canal pour le suivi ACK |

### Options de déploiement HEC distribué

1. Indexer direct (simple, pas de HA)
2. Heavy Forwarder (buffer + SSL termination)
3. Load Balancer → Heavy Forwarders → Indexers (recommandé pour la production)

---

## Module 17 — Fine-tune Inputs

### Phase input (sur le forwarder)

Traitée **avant** l'envoi à l'indexer. Configurée dans `inputs.conf` et `props.conf` côté forwarder.

Ce qu'on doit "bien faire" à cette étape :
- **host** · **sourcetype** · **source** · **index**

### props.conf côté forwarder — override du sourcetype

```ini
# props.conf sur le forwarder (ou dans l'app déployée)
[source::/var/log/myapp/app.log]
sourcetype = my_custom_sourcetype
```

> Splunk tente de **deviner automatiquement** le sourcetype si non spécifié. Cette auto-détection est fiable pour les formats connus (syslog, apache, etc.) mais peut être incorrecte pour des formats custom.

### Comportement du sourcetype automatique

Splunk inspecte le début du fichier et compare aux sourcetypes connus. Pour un répertoire avec des fichiers de types mixtes, l'auto-sourcetype est recommandé.

---

## Module 18 — Parsing Phase & Data Preview

### La phase de parsing

Traitée sur l'**indexer** (ou le Heavy Forwarder). Transforme le stream de bytes en **événements discrets**, chacun avec un timestamp.

Ce qu'on doit "bien faire" à cette étape :
- **Line breaking** (délimitation des événements)
- **Timestamp extraction**
- **Masquage des données sensibles** *(optionnel)*
- **Élimination d'événements** *(optionnel)*
- **Modification des champs meta** *(optionnel)*

### Création des événements (Event Creation)

**Étape 1 — Line breaking** (découpage en lignes) :
```ini
# props.conf
[my_sourcetype]
LINE_BREAKER = ([\r\n]+)    ← défaut : toute séquence de retours à la ligne
```

**Étape 2 — Line merging** (fusion de lignes en événements) :
```ini
SHOULD_LINEMERGE = true     ← défaut : les lignes sont fusionnées selon des règles
# Si chaque ligne est un événement séparé, désactiver pour meilleures perfs :
SHOULD_LINEMERGE = false
```

Autres règles de merging :
- `BREAK_ONLY_BEFORE` : nouvel événement uniquement si la ligne matche le pattern
- `BREAK_ONLY_BEFORE_DATE` : nouvel événement si la ligne commence par un timestamp
- `MUST_BREAK_AFTER` : forcer la fin d'un événement après ce pattern

### Extraction du timestamp

```ini
[my_sourcetype]
TIME_PREFIX    = \[                  ← pattern avant le timestamp
TIME_FORMAT    = %d/%b/%Y:%H:%M:%S  ← format strptime
MAX_TIMESTAMP_LOOKAHEAD = 40        ← nb de caractères à scanner
TZ             = UTC                ← timezone si non présente dans les logs
```

### Data Preview

Outil dans Splunk Web pour **valider** la création d'événements avant production.

Accessible via : Settings → Source Types → New Source Type, ou lors de l'ajout d'une data source.

Permet de voir en temps réel :
- Les event boundaries
- Les timestamps extraits
- Les champs parsés

---

## Module 19 — Manipulate Input Data

### Pourquoi modifier les données avant indexation

- **Confidentialité** : masquer numéros de carte bancaire, données patient (HIPAA), etc.
- **Routage** : envoyer certains événements vers des index spécifiques
- **Filtrage** : éliminer des événements inutiles (ne compte pas dans le quota de licence)

> ⚠️ Ces modifications changent le **`_raw`** avant indexation — les données indexées seront différentes de la source originale.

### Méthode 1 — Ingest Actions (Splunk 9.0+, Linux seulement)

Interface graphique dans Splunk Web. Crée des **rulesets** (un par sourcetype).

Actions disponibles par règle :
- **Mask** : remplace un pattern par un autre
- **Truncate** : tronque l'événement à partir d'un point
- **Route** : change l'index de destination
- **Eliminate** : supprime l'événement

> Ingest Actions doit être configuré sur un **Deployment Server dédié**.

### Méthode 2 — SEDCMD (masquage simple)

```ini
# props.conf
[my_sourcetype]
SEDCMD-mask_cc = s/\b\d{4}[- ]\d{4}[- ]\d{4}[- ]\d{4}\b/XXXX-XXXX-XXXX-XXXX/g
```

### Méthode 3 — TRANSFORMS (masquage/routing avancé)

```ini
# props.conf
[my_sourcetype]
TRANSFORMS-mask_ssn = mask_ssn_transform

# transforms.conf
[mask_ssn_transform]
REGEX  = \b(\d{3})-(\d{2})-(\d{4})\b
FORMAT = XXX-XX-$3          ← garde les 4 derniers chiffres
DEST_KEY = _raw             ← modifie le champ _raw
```

---

## Module 20 — Route Input Data

### Routing par index (per-event index routing)

```ini
# props.conf
[mysrctype]
TRANSFORMS-route = route_errors

# transforms.conf
[route_errors]
REGEX    = (Error|Warning)
DEST_KEY = _MetaData:Index
FORMAT   = security_index     ← redirige vers cet index
```

### Filtrage d'événements (null queue)

Les événements envoyés dans la **null queue** sont **supprimés** et **ne comptent pas** dans le quota de licence.

```ini
# props.conf
[WinEventLog:System]
TRANSFORMS = drop_noisy_events

# transforms.conf
[drop_noisy_events]
REGEX    = (?i)^EventCode=(592|593)
DEST_KEY = queue
FORMAT   = nullQueue           ← supprime ces événements
```

### Routing TCP vers différents groupes (via HF)

```ini
# props.conf sur le HF
[default]
TRANSFORMS-routing = route_errors

# transforms.conf
[route_errors]
REGEX    = error
DEST_KEY = _TCP_ROUTING
FORMAT   = errorGroup          ← groupe défini dans outputs.conf

# outputs.conf
[tcpout:errorGroup]
server = 10.1.1.200:9999

[tcpout:defaultGroup]
server = 10.1.1.250:9998
```

---

## Module 21 — Support Knowledge Objects

### Index-time vs Search-time field extractions

| | Index-time | Search-time |
|--|------------|-------------|
| **Quand** | Durant le parsing/indexation | Au moment de la recherche |
| **Où configuré** | `props.conf` + `transforms.conf` + `fields.conf` sur l'indexer/HF | `props.conf` + `transforms.conf` sur le Search Head |
| **Stockage** | Stocké dans l'index (augmente la taille de 2-5x) | Calculé à la volée, pas de surcoût de stockage |
| **Flexibilité** | Peu flexible (changement = ré-indexation) | Très flexible (changement instantané) |
| **Usage recommandé** | Sources fréquemment reconfigurées (ex: IIS) | La plupart des cas |

### Gérer les Knowledge Objects orphelins

Un KO est **orphelin** quand son propriétaire n'existe plus dans Splunk.

- Rechercher via Settings → Knowledge → Searches, reports, and alerts → filtrer par owner
- Réassigner via l'interface ou `./splunk cmd python reassign_ko.py`
- Pour les lab : **Lab 21** montre comment rechercher et réassigner un report orphelin

### Configured Indexed Field Extractions

Nécessite 3 fichiers :
1. `props.conf` : associe la transformation au sourcetype
2. `transforms.conf` : définit l'extraction
3. `fields.conf` : déclare les champs comme indexed (obligatoire)

```ini
# fields.conf
[my_field]
INDEXED = true
```

---

---

## Commandes CLI — référence complète

### Gestion du service

```bash
splunk start                              # Démarrer Splunk
splunk stop                               # Arrêter Splunk
splunk restart                            # Redémarrer Splunk
splunk status                             # Afficher le statut
splunk start --accept-license             # Démarrer en acceptant la licence automatiquement
splunk start --accept-license --answer-yes --no-prompt --seed-passwd <password>
```

### Informations système

```bash
splunk help                               # Aide générale
splunk help <object>                      # Aide sur un objet spécifique
splunk show default-hostname              # Nom d'hôte par défaut pour les inputs
splunk show servername                    # Nom du serveur Splunk
splunk show splunkd-port                  # Port splunkd (défaut: 8089)
splunk show web-port                      # Port Splunk Web (défaut: 8000)
splunk show config inputs                 # Configuration effective de inputs.conf
```

### Gestion des configurations (btool)

```bash
splunk btool check                        # Vérifier les erreurs de conf
splunk btool inputs list                  # Voir inputs.conf fusionné
splunk btool inputs list --debug          # Voir inputs.conf avec le fichier source
splunk btool inputs list monitor:///var/log --debug
splunk btool props list --debug           # Voir props.conf fusionné avec sources
splunk help btool                         # Aide btool
```

### Gestion des index

```bash
splunk add index <index_name>             # Créer un index
splunk add index <name> -app <app_name>   # Créer un index dans une app
splunk clean eventdata -index <name>      # Supprimer les données d'un index
splunk clean eventdata                    # Supprimer les données de tous les index
splunk clean userdata                     # Supprimer les saved searches, reports...
splunk clean all                          # Tout supprimer (DANGER !)
```

### Gestion des licences

```bash
splunk add licenses <keyfile>             # Ajouter une licence depuis un fichier
```

### Gestion des forwarders

```bash
# Sur l'indexer — configurer le port de réception
splunk enable listen <port>               # Activer la réception S2S
splunk display listen                     # Afficher le port de réception actuel

# Sur le forwarder — configurer la destination
splunk add forward-server <host:port>     # Ajouter un indexer comme destinataire
splunk remove forward-server <host:port>  # Supprimer un destinataire
splunk list forward-server                # Lister les destinataires configurés
```

### Gestion du Deployment Server

```bash
# Sur le client (forwarder)
splunk set deploy-poll <DS_host:port>     # Configurer le DS (port 8089)
splunk show deploy-poll                   # Voir la config deployment client
splunk disable deploy-client              # Désactiver le deployment client

# Sur le DS
splunk reload deploy-server               # Recharger la config du DS
splunk list deploy-clients                # Lister les clients connectés
```

### Gestion des inputs

```bash
# Ajouter un monitor input
splunk add monitor /var/log/access.log
splunk add monitor /var/log/ -index web -sourcetype access_combined
./splunk add monitor /opt/log/www1/access.log -index itops \
  -sourcetype access_combined_wcookie -host splunk01

# Configurer le serveur
./splunk set default-hostname uf01        # Changer le hostname par défaut
./splunk set servername uf01              # Changer le nom du serveur
```

### Gestion des users

```bash
splunk edit user <user> -locked-out false -auth admin:<password>   # Débloquer un user
```

### Fishbucket

```bash
splunk clean eventdata -index _thefishbucket           # Réinitialiser le fishbucket
splunk cmd btprobe -d $SPLUNK_DB/fishbucket/splunk_private_db  # Inspecter le fishbucket
```

### Diagnostics

```bash
splunk diag                               # Générer un bundle de diagnostique
splunk cmd btprobe -d <path>              # Inspecter un index ou le fishbucket
```

### Scripts

```bash
splunk cmd <script>                       # Exécuter un script dans le contexte Splunk
# Exemple :
splunk cmd SPLUNK_HOME/etc/apps/my_app/bin/myscript.sh
```

---

## Fichiers de configuration — référence

### inputs.conf

```ini
# Monitor fichier
[monitor:///var/log/secure]
disabled    = false
sourcetype  = linux_secure
index       = security
host        = webserver01
blacklist   = \.gz$
whitelist   = \.log$
ignoreOlderThan = 7d          # Ignorer les fichiers non modifiés depuis 7 jours

# Network TCP
[tcp://9001]
connection_host = dns
sourcetype      = syslog

# Network UDP
[udp://514]
connection_host = ip
sourcetype      = syslog

# Script
[script://./bin/myscript.sh]
interval   = 60
sourcetype = my_metrics

# Réception S2S (sur indexer)
[splunktcp://9997]
```

### outputs.conf

```ini
[tcpout]
defaultGroup = my-indexers

[tcpout:my-indexers]
server      = idx1:9997, idx2:9997
compressed  = true
useSSL      = true
useACK      = true

[tcpout-server://idx1:9997]
```

### props.conf

```ini
[my_sourcetype]
# Line breaking
LINE_BREAKER        = ([\r\n]+)
SHOULD_LINEMERGE    = false

# Timestamp
TIME_PREFIX         = \[
TIME_FORMAT         = %d/%b/%Y:%H:%M:%S
MAX_TIMESTAMP_LOOKAHEAD = 40
TZ                  = UTC

# Masquage SEDCMD
SEDCMD-mask_cc      = s/\b\d{4}[- ]\d{4}[- ]\d{4}[- ]\d{4}\b/XXXX-XXXX-XXXX-XXXX/g

# Lier une transformation
TRANSFORMS-route    = my_routing_transform
TRANSFORMS-mask     = my_mask_transform
```

### transforms.conf

```ini
# Routing vers un index
[my_routing_transform]
REGEX    = (Error|Warning)
DEST_KEY = _MetaData:Index
FORMAT   = security

# Drop vers null queue
[drop_events]
REGEX    = (?i)debug
DEST_KEY = queue
FORMAT   = nullQueue

# Masquage _raw
[my_mask_transform]
REGEX    = (\d{3})-(\d{2})-(\d{4})
FORMAT   = XXX-XX-$3
DEST_KEY = _raw

# Routing TCP
[tcp_routing]
REGEX    = error
DEST_KEY = _TCP_ROUTING
FORMAT   = errorGroup
```

### indexes.conf

```ini
[my_index]
homePath   = $SPLUNK_DB/my_index/db
coldPath   = $SPLUNK_DB/my_index/colddb
thawedPath = $SPLUNK_DB/my_index/thaweddb

# Taille
maxDataSize          = auto_high_volume   # auto=750MB, auto_high_volume=10GB
maxTotalDataSizeMB   = 307200             # 300 GB total
homePath.maxDataSizeMB = 100000
coldPath.maxDataSizeMB = 300000

# Rétention temporelle
frozenTimePeriodInSecs = 31536000         # 1 an

# Buckets
maxHotBuckets   = 3
maxWarmDBCount  = 300
maxHotSpanSecs  = 86400                   # 1 jour
```

### serverclass.conf (Deployment Server)

```ini
[serverClass:linux_forwarders]
whitelist.0   = *linux*
appList       = my_linux_inputs_app, base_outputs_app
restartSplunkd = true

[serverClass:windows_forwarders]
whitelist.0   = *win*
appList       = my_windows_inputs_app
```

---

## Ports réseau par défaut

| Port | Usage | Composant |
|------|-------|-----------|
| **8000** | Splunk Web (HTTP) | Search Head / Standalone |
| **8065** | Web app-server proxy | Search Head |
| **8088** | HTTP Event Collector (HEC) | Indexer / HF |
| **8089** | splunkd management (REST API, CLI, DS communication) | Tous les composants |
| **8191** | KV Store (MongoDB interne) | Search Head |
| **9997** | S2S (Splunk-to-Splunk) — port de réception des forwarders | Indexer |

> ⚠️ Les ports S2S, réseau/HTTP inputs, réplication d'index et de search **n'ont pas de valeur par défaut** — ils doivent être explicitement configurés.

---

## Pièges et best practices

### Configuration

- Ne **jamais modifier** `SPLUNK_HOME/etc/system/default/` — ces fichiers sont écrasés à chaque mise à jour
- Créer une **app dédiée** (ex: `DC_app`) pour gérer les settings système à déployer via DS
- Les settings **index-time** ne peuvent pas être overridés dans une app sur un indexer en production — modifier `system/local` ou utiliser une app déployée par le DS
- Toujours utiliser `btool --debug` pour vérifier quelle couche de conf est active

### Données et index

- **Tester d'abord** sur un serveur dev/test avec la même version que la prod
- Les données mal indexées **ne peuvent pas être corrigées** — elles doivent vieillir jusqu'à expiration
- `clean eventdata` est **irréversible**
- La réinitialisation du fishbucket provoque la **ré-indexation** de tous les fichiers surveillés
- Séparer les données dans **différents index** selon les besoins de rétention et d'accès

### Sécurité

- Conserver un **compte failsafe natif Splunk** même avec SAML/LDAP activé
- Désactiver le port **8089** sur les UF en production si non nécessaire
- Ne pas faire tourner Splunk en **root** — utiliser un compte dédié
- Synchroniser les horloges (**NTP**) sur tous les serveurs — critique pour les corrélations

### Licences

- Les événements dans la **null queue** ne comptent **pas** dans le quota journalier
- Les données de summary index et réplication ne comptent **pas** non plus
- Configurer une alerte MC sur l'approche du quota pour éviter les violations

### Inputs

- Toujours spécifier `sourcetype`, `index`, `host` explicitement — ne pas compter sur l'auto-détection en production
- Préférer **TCP** à **UDP** pour les données importantes (UDP ne garantit pas la livraison)
- Utiliser **HEC avec ACK** pour les données critiques (confirmation d'indexation)
- Les scripts ne sont exécutés **que** depuis les répertoires autorisés (bin/)

### Forwarding

- Le UF est limité à **256 KBps par défaut** — augmenter si nécessaire avec `maxKBps` dans outputs.conf
- Le Deployment Server doit être sur un **serveur dédié** avec une licence Enterprise
- Le DS communique via le port **8089** — vérifier les firewalls

---

*Bonne révision — Good luck pour la certification Splunk Enterprise Admin !*
