
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






