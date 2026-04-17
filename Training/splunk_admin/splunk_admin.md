# Day 1 

### Deploy splunk 

4 stages : Inout any text data / Parse the date into events / Search and report 

Simple Test server : 1 server for searching / indexing / Parsing / input

parsing : ou l'event s'arrete et ou ke suivant commence pour etre lisible sur le search head

universal forwarder a besoin de fichier inputs (ce que je récupère) et outputs (ou les envoyer)

Monitoring console, vérifier etat de santé des serveurs, l'état des indexers, vérifier les envois des universal forwarder 
si cluster , besoin d'un cluster manager (qui ne fera pas parti du cluster)

default port : 
splunkd 8089 (The splunk process)
Splunk web 8000
Web app server proxy 8065
KV strore 8191
(modifier port Settings > server settings > app server port)

installation splunk sur linux, vérifier si le boot-start est activé pour que splunk redemarre tout seul en cas de redemarrage du serveur 
init.d
splunk enable boot-start -user username
systemd
splunk enable boot-start -user username -systemd-managed 1

Best securtity practice : 
Chanfe admin password
User appropriate roles
disable port 8089 on splunk UF
Use host FW
Secure splunk to splunk communication 
Replace the default certificates


smart store : équivalent à un S3 dans le cloud, on retire les indexers et les stock dans le cloud

searchpeer : équivalent à indexer mais il travaille avec le search head

Splunk diag : créé un tar gz avec info sur conf (attention ne prend pas les logs / events) 
montrera l'index mais pas le contenu 

  après restart, les donnees hot deviennent warm
  
The KV store stores your data as key-value pairs in collections ( Les recherches font souvent référence à un fichier csv ou à un KV store)

## Configuration files
toute conf est modifiable via la GUI si possible , CLI et API rest
([stanza] paramètre)
search head :
props.conf > search time Field Extraction , lookups and so on, gère les objetf, alias, champs calculé etc 
Indexer
pros.conf -> paramètre depend du serveur sur lequel il est (parsing) PAGE 92
inputs.conf > what date is ollected which ports to listen
Forwarder 
outputs.conf where to forward data 
props.conf limited parsing 
inputs.conf what data is collected

/!\les fichiers dans le default ne doivent pas etre midifiés, les patchs remettent à 0 ces repertoires

mocro fonctionnent uniquement sur le Search head
Ordre de prio :
system / local 
apps / appname /local 
apps / appname / default
system / default

les apps sont classées par lexico graphique exemple U est avant v 

/!\ ne rien mettre dans le system/local 

## configuration validation tool
Run splunk btool check each time splunk starts

## Track configuration 
stores changes in the _configtracker index


## App
Collection and configuration files scripts web assets

renommer une app en .old ne sert a rien, cela renommera juste l'app si besoin de conserver une app sans supprimer la changer de repertoire 

## Create Index

on créé un fichier, jamais sur GUI (sauf en cas de test)
un index c'est un fichier plat stocké en sous répertoire dans var lib splunk (Splunk_DB)
on détruit pas des events mais le bucket (politique de rétention) 
les buckets chaud et tiende partagent un même repetoire


# Day 2

useACK attente du l'ACK de l'indexer 
 Dans indexes.conf, préciser les volumes
/!\ si pas volume, splunk ne peut pas monitorer le filesystem

## Backup
backup du ETC  +++ stocké dans SPLUNK_HOME/var/lib/splunk

## move an index
un index peut etre déplacer mais penser a maj indexers.conf
les events dans l'index ne peuvent pas etre supprimé
 | delete , laisse les events en place mais ne les montre plus au search 
    exemple : index=soc host=myds_temp souce=access.log | delete
      si mauvaise manip, voir avec le support car les données sont toujours existantes

## Fishbucket
Allows Splunk to track monitored input files
Contains file metadata which identifies a pointer to the file, and a pointer to where Splunk last read the file
l'option continuously Monitor permet de garder une continuité dans le monitoring, en cas de coupure il reprendre à partir du fishbucket, reprendre ou il s'est arreté 

reset du fish bucket avec la commande splunk cmd btprobe –d SPLUNK_DB/fishbucket/splunk_private_db --file <source> --reset
(stoper splunk puis relancer après)

frozen bucket -> splunk rebuild <Thawed_Path> pour remettre les données hors frozen

## Users and Roles 
La capability d'acceleration permet de créer les objets accélérés (qui vont écrire dans les indexes). 
Pour accéder à l'information accélérée, il suffit d'avoir la capacité de lecture sur l'index .
(toutes recherches doient mentionner un index)

disk space limit : dans la création d'un nouveau role, on a la possibilité de mettre un "disk space limit" 
lors d'un search, la requete est stockée dans via un job sur le drive ( par défaut stocké pendant 10min, pour un report, 2 prochaines exécutions, modifiable dans le SH)

le role edit roles grantable (donne roles que j'ai déjà) et edit user permet de donner la possiblité de gérer les users

l'option -locked-out (en cli) permet de ne pas bloquer en cas de failled pwd
splunk edit user <locked_user> -locked-out false –auth <admin_user:password>


## Get Data Into Splunk

Input >     Parsing > Indexing > indexes < searching
Forwarder  |              Indexer          | Search head

input type :
Files / directories
Netword data 
script output
Linux and windows logs
Http 
......

You can add data inputs with :
Apps and add ons
SPlunk web
CLI
Editing inputs.conf

### Metadata Settings
| host |>| Where abd event originates |
| source |>| source file, stream or input of an event |
| Sourcetype |>| Format and category of the data input |
| Index |>| where data is stored by splunk |

A tsidx file associates each unique keyword in your data with location references to events

si symbole = dans l'event alors sorti par le search head (key value pair)

les HEC permettent d'encapsuler en cas de flux hors tcp

## HF UF
| Critère                          | Universal Forwarder (UF)                                                                 | Heavy Forwarder (HF)                                                                 |
|----------------------------------|------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| Usage principal                  | Idéal pour la plupart des cas (collecte de logs, forwarder intermédiaire)               | Utilisé pour des cas spécifiques et avancés                                         |
| Infrastructure                  | Léger, faible empreinte sur les serveurs de production                                  | Fonctionne généralement sur des serveurs dédiés                                     |
| Performance / Bande passante     | Moins de bande passante, traitement plus rapide                                         | Peut augmenter le trafic réseau                                                     |
| Routage des données              | Routage simple ou duplication vers plusieurs indexers                                   | Routage complexe au niveau des événements                                           |
| Filtrage des données             | Ne supporte pas le filtrage basé sur des expressions régulières                         | Peut anonymiser ou masquer les données avant envoi                                  |
| Cas d’utilisation spécifiques    | Usage général                                                                          | Requis pour certains apps, add-ons ou inputs (ex: HEC, DB Connect)                 |
| Fonctionnalités supplémentaires  | Limité                                                                                | Fournit Splunk Web et un environnement Python contrôlé                             |


Check current splunktcp settings in inputs.conf using the btool command.
./splunk btool inputs list splunktcp --debug

to set a listening port : ./splunk enable listen 9997

## load balancing
bascule quand end of file ou 30 sec (by default) 


# Day 3
On peut créer des groupes pour le DS deploie à des groupes séparé (exemple linux | windows)

## DS 
Le DS pousse les conf sur les serveurs. C'est le client requete le DS.
exemple : deployer les apps vers les indexers.

(dans le dashboard du DS, il s'appelle Forwarder Management, voir dans SETTINGS)

Conf:
Maps clients to apps : /etc/apps/<app>/local/serverclass.conf
App repo : /etc/deployme,t-apps/<app>/local
App/config to deploy : outputs.conf, inputs.conf, etc..

Le server class est le mapping entre serveurs et apps 
exemple : server class W -> APP Wini -> Windows 1
                                     -> Windows AD 


## Monitor input 
utilisation possible de * et ... (pour tous les répertoires) dans le fichier de conf ex : 
[monitor:///var/log/www1/secure.log]
[monitor:///var/log/www*/secure.*]
[monitor:///var/log/.../secure.*]

possibilité de mettre des blacklist et whitelist (utilisation de ... et * impossible)
ex : 
[monitor:///var/log/www1/]
whitelist = \.log$


## Network Inputs

## Scripted inputs 
tourne sur le serveur source (UF)
sortie script ajouter dans la recherche sur le SH : "| multikv" (/!\ attention les entetes doivent etre dans le même event)
ajouter "| transaction" avant multikv  pour regrouper les events 


## Agentless Inputs

Configure a HTTP Event collector agentless inputs (HEC)
Send events to splunk without the use of forwarders (activate on HF)

## Fine-tune Inputs 

index time process :
Input base handled at the source (usually a forwarder)
Parsing phase :
handled by indexers (or heavy forwarders)
Indexing phase : 
Handled by indexers 

Input       ->   parsing -> licencse meter -> indexing    ->search-> web
Forwarder                         Indexers                      Search head

le fichier props.conf est utilisé dans toutes les phases
on retrouve en stanza par exemple : 
source 
host
sourcetype

(le event breaker sert pour le loadbalancing)

## Parsing Phase and data Preview 

Le parsing a lieu sur le parser 
Inputs     -> parsing -> license meter -> Indexing
Forwarder                 Indexer
Le parsing créé, modifie et redirige les events

Le HF découpe : il identifie la limite entre chaque event à l'aide du line breaker 
attention une date est considérée comme un 'break' (il faudra ajouter : BREAK_ONLY_BEFORE_DATE = true)


# Day 4

## Ingest Actions
Can mask, truncate, route or eliminate data (rulesets in Splunk Web)

CM automatically distributes to Indexers

Order of Ingest Action and Classic Rules :
1. Heavy Forwarder Classic transforms
2. Heavy Forwarder Ingest Action ruleset transforms
3. Indexer Classic transforms
4. Indexer Ingest Action ruleset transforms



_____________________________________________________________________________________________
new format 
_____________________________________________________________________________________________




# Splunk Administration - Full Study Notes

## Day 1: Deployment & Fundamentals

### Splunk Pipeline (4 Stages)
1. **Input**: Récupération de n'importe quelle donnée texte.
2. **Parsing**: Découpage en évènements (où l'event s'arrête et où le suivant commence pour être lisible).
3. **Indexing**: Stockage des données sur disque.
4. **Search & Report**: Requêtes via l'interface.

### Server Architectures
* **Simple Test Server**: Une seule instance pour Search / Indexing / Parsing / Input.
* **Universal Forwarder (UF)**: Nécessite des fichiers `inputs.conf` (ce que je récupère) et `outputs.conf` (où les envoyer).
* **Monitoring Console (MC)**: Vérifier l'état de santé, les indexeurs et les envois des UF.
* **Cluster Manager (CM)**: Nécessaire si cluster d'indexeurs. **Attention** : il ne doit pas faire partie du cluster lui-même.
* **Search Peer**: Équivalent à un indexeur, mais vu sous l'angle du Search Head (SH).
* **Smart Store**: Équivalent à un S3 dans le cloud. On retire les données des indexeurs pour les stocker dans le cloud.

### Ports par défaut
* **8089**: `splunkd` (Le processus Splunk principal).
* **8000**: Splunk Web.
* **8065**: Web app server proxy.
* **8191**: KV Store.
* *Modification via : Settings > Server Settings > App server port.*

### Installation Linux & Sécurité
* **Boot-start**: Vérifier s'il est activé pour redémarrer Splunk automatiquement.
  * `init.d` : `splunk enable boot-start -user username`
  * `systemd` : `splunk enable boot-start -user username -systemd-managed 1`

**Best Practices Sécurité :**
* Changer le mot de passe admin.
* Utiliser les rôles appropriés.
* Désactiver le port **8089** sur les Splunk UF.
* Utiliser le Firewall de l'hôte (Host FW).
* Sécuriser les communications Splunk-to-Splunk.
* Remplacer les certificats par défaut.

### Stockage & Diagnostics
* **Splunk Diag**: Crée un `.tar.gz` avec les infos de conf. **Attention** : ne prend pas les logs ni les évènements. Montrera l'index mais pas le contenu.
* **Buckets**: Après un restart, les données **Hot** deviennent **Warm**.
* **KV Store**: Stocke les données sous forme de paires clé-valeur dans des collections (souvent utilisé pour remplacer des fichiers CSV).

---

## Day 2: Configuration & Data Management

### Configuration Files logic
* Toute conf est modifiable via : **GUI** (si possible), **CLI** ou **API REST**.
* Format : `[stanza] paramètre = valeur`.

**Rôles des fichiers selon le composant :**
* **Search Head (SH)**: 
  * `props.conf` : Search-time Field Extraction, lookups, alias, champs calculés.
* **Indexer**: 
  * `props.conf` : Paramètres de parsing (dépend du serveur).
  * `inputs.conf` : Quelles données collecter et quels ports écouter.
* **Forwarder**:
  * `outputs.conf` : Où envoyer les données.
  * `inputs.conf` : Quelles données collecter.
  * `props.conf` : Parsing limité.

> [!WARNING]
> **Important** : Les fichiers dans `/default/` ne doivent jamais être modifiés (les patchs les réinitialisent).
> Les **Macros** fonctionnent uniquement sur le Search Head.

**Ordre de priorité (Precedence) :**
1. `system / local`
2. `apps / appname / local`
3. `apps / appname / default`
4. `system / default`
*Les apps sont classées par ordre lexicographique (U est avant V).*
* **Conseil** : Ne rien mettre dans `system/local`.

### Maintenance & Index
* **Btool**: Lancer `splunk btool check` à chaque démarrage.
* **Config Tracker**: Suit les changements dans l'index `_configtracker`.
* **Apps**: Collection de fichiers de conf, scripts et web assets. Renommer une app en `.old` ne sert à rien (elle sera toujours lue), il faut la déplacer hors du répertoire.
* **Création d'Index**: Toujours via fichier de conf (jamais via GUI sauf test). C'est un fichier plat dans `var/lib/splunk` (`SPLUNK_DB`).
* **Rétention**: On détruit le **bucket**, pas l'évènement individuel. Les buckets *Hot* et *Warm* partagent le même répertoire.

### Backup & Fishbucket
* **Backup**: Sauvegarder le dossier `ETC`.
* **Move Index**: Possible, mais il faut mettre à jour `indexes.conf`.
* **Delete**: La commande `| delete` laisse les évènements en place mais les cache au search. Pour récupérer une mauvaise manip, voir avec le support.
* **Fishbucket**: Permet de tracker les fichiers monitorés (pointeurs).
  * Option `continuously Monitor` : Permet de reprendre là où Splunk s'est arrêté après une coupure.
  * Reset : `splunk cmd btprobe –d SPLUNK_DB/fishbucket/splunk_private_db --file <source> --reset` (Stopper Splunk avant).
* **Frozen**: Pour remettre des données Frozen en ligne, utiliser `splunk rebuild <Thawed_Path>`.

### Users & Roles
* **Capabilities**: L'accélération permet de créer des objets accélérés. Pour lire, il suffit d'avoir le droit de lecture sur l'index.
* **Disk Space Limit**: Dans le rôle, limite l'espace pour les jobs de recherche (stockés par défaut 10min).
* **Roles Edit**: `edit_roles_grantable` (donne les rôles que j'ai déjà) et `edit_user`.
* **Unlock User**: `splunk edit user <locked_user> -locked-out false –auth <admin_user:password>`.

---

## Day 3: Getting Data In (GDI)

### Metadata & Pipeline
* **Metadata**: Host (origine), Source (fichier/flux), Sourcetype (format), Index (lieu de stockage).
* **TSIDX**: Fichier qui associe chaque mot-clé unique à l'emplacement des évènements.
* **Load Balancing**: Bascule quand "End of File" est atteint ou après 30 sec par défaut.

### HF vs UF Comparison
| Critère | Universal Forwarder (UF) | Heavy Forwarder (HF) |
| :--- | :--- | :--- |
| **Usage** | Idéal pour 90% des cas | Cas spécifiques et avancés |
| **Infrastructure** | Léger, faible empreinte | Serveur dédié (Splunk complet) |
| **Bande passante** | Traitement rapide | Peut augmenter le trafic |
| **Filtrage** | Pas de Regex | Supporte Regex et Anonymisation |
| **Fonctionnalités** | Limité | Splunk Web + Python |

* Commande Btool pour vérifier : `./splunk btool inputs list splunktcp --debug`
* Écoute : `./splunk enable listen 9997`

### Deployment Server (DS)
* Pousse les confs aux clients (c'est le client qui requête le DS).
* Dashboard : **Forwarder Management** dans Settings.
* **Server Class**: Le mapping entre serveurs et apps.
* Chemins :
  * Client/App mapping : `/etc/apps/<app>/local/serverclass.conf`
  * App repo : `/etc/deployment-apps/`

### Inputs avancés
* **Monitor**: Supporte `*` (un niveau) et `...` (récursif).
  * Exemple : `[monitor:///var/log/.../secure.*]`
* **Blacklist/Whitelist**: Regex autorisées, mais `...` et `*` interdits dedans.
* **Scripted Inputs**: Tournent sur le serveur source (UF).
  * Sur le SH : ajouter `| multikv` (les entêtes doivent être dans le même event).
  * Utiliser `| transaction` avant multikv pour regrouper si besoin.
* **HEC (HTTP Event Collector)**: Agentless. Pour envoyer des données sans forwarder (à activer sur HF).

---

## Day 4: Parsing & Ingest Actions

### Parsing Phase
* Se passe sur l'Indexer ou le HF.
* **Line Breaker**: Identifie la limite entre chaque évènement.
* **Date**: Si une date est un "break", utiliser `BREAK_ONLY_BEFORE_DATE = true`.
* **Event Breaker**: Sert spécifiquement pour le load balancing.

### Ingest Actions
* Permet de masquer, tronquer, router ou supprimer des données (Rulesets dans Splunk Web).
* Le Cluster Manager (CM) les distribue automatiquement aux Indexers.

**Ordre de priorité des règles (Ingest vs Classic) :**
1. Heavy Forwarder : **Classic transforms**
2. Heavy Forwarder : **Ingest Action ruleset**
3. Indexer : **Classic transforms**
4. Indexer : **Ingest Action ruleset**






