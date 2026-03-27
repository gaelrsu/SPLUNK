# Using Fields

## Architecture Splunk — les composants essentiels

**Forwarder (collecte)**  
Collecte et transmet les données. Universal Forwarder = léger, recommandé. Heavy Forwarder = parsing local possible.

**Indexer (stockage)**  
Reçoit, parse, indexe les données dans des buckets. Effectue les recherches sur ses index.

**Search Head (UI)**  
Interface web pour les utilisateurs. Distribue les recherches aux indexers et agrège les résultats.

---

## Flux de données — de la source à l'index

1. **Input** — Splunk consomme des données : fichiers, syslog, scripts, API...
2. **Parsing** — Découpe en événements, extrait le timestamp, identifie le sourcetype
3. **Indexing** — Compression, écriture dans les buckets (hot → warm → cold → frozen)
4. **Searching** — L'utilisateur interroge via SPL, Splunk retourne les événements

---

## Recherche SPL


index=web sourcetype=access_combined
status=404 uri="/login*"
| stats count by clientip
| sort -count
| head 10


| Partie | Rôle |
|--------|------|
| index=web sourcetype=... | Critères de recherche (filtrage sur les index/champs) |
| status=404 | Filtre sur un champ extrait ou par défaut |
| uri="/login*" | Filtre avec wildcard sur un champ |
| \| stats count by | Pipe : passe les résultats à la commande suivante |
| \| sort -count | Tri décroissant (- = décroissant, sans = croissant) |
| \| head 10 | Limite à 10 premiers résultats |

---

## Opérateurs booléens et wildcards

### Booléens — MAJUSCULES obligatoires


error AND login
error OR warning
NOT debug
(error OR fail) AND login


### Wildcards


fail* → fail, failed, failure
error → contient "error"
192.168.* → toute IP 192.168.x.x
user_? → user_a, user_1 (1 char)


---

## Champs par défaut

| Champ | Description | Exemple |
|------|------------|---------|
| _time | Timestamp de l'événement (Unix epoch) | 1711234567 |
| _raw | Données brutes de l'événement | Texte original du log |
| host | Hôte source des données | webserver-01 |
| source | Chemin du fichier ou URI source | /var/log/syslog |
| sourcetype | Format/type des données | access_combined, syslog |
| index | Index où l'événement est stocké | main, web, security |
| splunk_server | Serveur Splunk ayant indexé l'événement | splunk-idx-01 |

---

## Search modes — impact sur les performances

| Mode | Description | Usage |
|------|------------|------|
| Fast mode | Moins de champs extraits, plus rapide | Recherches larges, monitoring rapide |
| Smart mode | Par défaut, équilibre | Usage quotidien |
| Verbose mode | Extrait tous les champs, plus lent | Investigation, débogage |

---

## Commandes de filtrage et transformation essentielles

| Commande | Syntaxe | Rôle |
|----------|--------|------|
| search | `| search status=200` | Filtre supplémentaire après un pipe |
| where | `| where count > 100` | Filtre avec expressions |
| fields | `| fields host, status, uri` | Conserve ou supprime des champs |
| rename | `| rename clientip AS "IP Source"` | Renomme un champ |
| table | `| table _time, host, status` | Affiche une table |
| head | `| head 20` | N premiers résultats |
| tail | `| tail 10` | N derniers résultats |
| sort | `| sort -count, +host` | Tri |
| dedup | `| dedup host` | Supprime doublons |
| eval | `| eval total=price*qty` | Crée/modifie champ |
| rex | `| rex "(?P<user>\w+)"` | Extraction regex |

---

## fields — conserver vs supprimer

**Inclusion**

| fields host status uri


**Exclusion**

| fields - _raw splunk_server


---

## eval — fonctions importantes

### Calculs

| eval total = price * quantity
| eval margin = round((revenue - cost) / revenue * 100, 2)


### Conditions

| eval severity = if(status >= 500, "critical", "normal")
| eval label = case(status==200,"OK", status==404,"Not Found", 1==1,"Other")


### Chaînes

| eval fullname = first . " " . last
| eval domain = lower(host)
| eval short_url = substr(uri, 1, 20)


### Temps

| eval readable_time = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval ts = strptime("2024-01-15", "%Y-%m-%d")


---

## rex — extraction par regex


| rex "user=(?P<username>\w+)"
| rex field=uri "/api/(?P<endpoint>[^/]+)"
| rex mode=sed "s/\d{4}-\d{4}-\d{4}-\d{4}/XXXX-XXXX-XXXX-XXXX/g"


---

## stats — agrégations essentielles


| stats count
| stats count by host, status
| stats sum(bytes) AS total_bytes by host
| stats avg(response_time) by uri
| stats dc(user) AS unique_users
| stats values(status) by host


| Fonction | Description |
|----------|------------|
| count | Nombre d'événements |
| dc() | Valeurs distinctes |
| sum/avg/min/max | Agrégations |
| values() | Valeurs uniques |
| list() | Avec doublons |
| earliest/latest | Ancien/récent |
| perc95 | Percentile |
| range | Écart |

---

## chart vs timechart


| chart count over host by status
| timechart count by status


- **chart** → axe libre  
- **timechart** → axe = _time

---

## top et rare


| top clientip
| rare clientip


Sortie :
- valeur
- count
- percent

---

## Types de champs Splunk

| Type | Description | Exemple |
|------|------------|---------|
| Default fields | Présents sur tous les événements | `_time`, `host` |
| Extracted fields | Extraits à la recherche | `status`, `clientip` |
| Indexed fields | À l’indexation | transforms.conf |
| Calculated fields | Via eval | `total=price*qty` |
| Lookup fields | Enrichis via table | `country` |

---

## Selected vs Interesting fields

**Selected fields**  
Toujours visibles (host, source, sourcetype)

**Interesting fields**  
Présents ≥ 20% des événements

---

## Tags et Event Types

**Tags**

tag=web_server


**Event Types**

eventtype=http_errors


---

## Field extraction — méthodes

| Méthode | Description | Usage |
|--------|------------|------|
| IFX | Interface graphique | Simple |
| Regex | Expression régulière | Complexe |
| Delimiters | CSV, pipe | Structuré |

---

## Types de visualisations et prérequis

| Viz | Prérequis des données | Cas d'usage |
|-----|----------------------|-------------|
| Bar chart | stats/chart | Comparaison |
| Line chart | timechart | Tendances |
| Area chart | idem | Volume |
| Pie | stats count | Répartition |
| Scatter | 2 champs num | Corrélation |
| Bubble | 3 champs | Volume |
| Single value | 1 valeur | KPI |
| Gauge | seuils | Alertes |
| Map | champ geo | Distribution |
| Cluster | lat/long | GPS |

---

## Dashboards — concepts clés

**Panels**
- Search
- Visualization
- Single value
- Table
- Text/HTML

**Permissions**
- Private
- App
- Global

---

## Rapports (Reports)

| Paramètre | Description |
|----------|------------|
| Schedule | Planification |
| Time range | Fenêtre |
| Acceleration | Performance |
| Permissions | Accès |
| Embed | Intégration |

---

## Alertes — types et actions

### Types
- Scheduled
- Real-time

### Conditions
- Per-result
- Count > N
- dc(host) > N

### Actions
- Email
- Script
- Webhook
- Log

**Note** : éviter le real-time en production.

---

## Lookups — enrichir les données

| Commande | Rôle |
|----------|------|
| lookup | Enrichir |
| inputlookup | Charger |
| outputlookup | Sauvegarder |

### Syntaxe


| lookup geo_lookup clientip
| inputlookup users.csv
| outputlookup stats.csv


---

## Types de lookup tables

| Type | Description | Usage |
|------|------------|------|
| CSV | Fichier statique | Référentiel |
| KV Store | Base interne | Dynamique |
| External | Script | API |
| Geospatial | KMZ | Cartes |
