# Fiches de révision — Splunk Core Certified User (SPLK-1001)

---

## Sommaire

1. [Architecture & Composants](#1-architecture--composants)
2. [Flux de données](#2-flux-de-données)
3. [Recherche SPL — Anatomie](#3-recherche-spl--anatomie)
4. [Opérateurs booléens & Wildcards](#4-opérateurs-booléens--wildcards)
5. [Champs par défaut](#5-champs-par-défaut)
6. [Search modes](#6-search-modes)
7. [Commandes de filtrage & transformation](#7-commandes-de-filtrage--transformation)
8. [Commande `fields`](#8-commande-fields)
9. [Commande `eval`](#9-commande-eval)
10. [Commande `rex`](#10-commande-rex)
11. [Commande `stats`](#11-commande-stats)
12. [Commandes `chart` vs `timechart`](#12-commandes-chart-vs-timechart)
13. [Commandes `top` et `rare`](#13-commandes-top-et-rare)
14. [Types de champs](#14-types-de-champs)
15. [Selected vs Interesting fields](#15-selected-vs-interesting-fields)
16. [Tags & Event Types](#16-tags--event-types)
17. [Field extraction — méthodes](#17-field-extraction--méthodes)
18. [Visualisations](#18-visualisations)
19. [Dashboards](#19-dashboards)
20. [Rapports (Reports)](#20-rapports-reports)
21. [Alertes](#21-alertes)
22. [Lookups](#22-lookups)

---

## 1. Architecture & Composants

| Composant | Rôle |
|-----------|------|
| **Universal Forwarder** | Collecte et transmet les données. Léger, recommandé en production. |
| **Heavy Forwarder** | Collecte + parsing local possible. Plus gourmand en ressources. |
| **Indexer** | Reçoit, parse, indexe les données dans des buckets. Effectue les recherches sur ses index. |
| **Search Head** | Interface web pour les utilisateurs. Distribue les recherches aux indexers et agrège les résultats. |

> **À retenir :** En mode **distribué**, chaque composant est sur une machine distincte. En mode **standalone**, tout est sur un seul serveur.

---

## 2. Flux de données

```
Source → Forwarder → Indexer → Search Head
```

| Étape | Description |
|-------|-------------|
| **1. Input** | Splunk consomme des données : fichiers, syslog, scripts, API... |
| **2. Parsing** | Découpe en événements, extrait le timestamp, identifie le sourcetype |
| **3. Indexing** | Compression, écriture dans les buckets : `hot → warm → cold → frozen` |
| **4. Searching** | L'utilisateur interroge via SPL, Splunk retourne les événements |

> **Buckets :** `hot` (actifs) et `warm` (récents) = disque rapide. `cold` = disque lent. `frozen` = archivés ou supprimés.

---

## 3. Recherche SPL — Anatomie

```spl
index=web sourcetype=access_combined
  status=404 uri="/login*"
| stats count by clientip
| sort -count
| head 10
```

| Partie | Rôle |
|--------|------|
| `index=web sourcetype=...` | Critères de recherche — filtrage sur les index/champs |
| `status=404` | Filtre sur un champ extrait ou par défaut |
| `uri="/login*"` | Filtre avec wildcard sur un champ |
| `\| stats count by` | Pipe : passe les résultats à la commande suivante |
| `\| sort -count` | Tri décroissant (`-` = décroissant, sans signe = croissant) |
| `\| head 10` | Limite à 10 premiers résultats |

---

## 4. Opérateurs booléens & Wildcards

### Booléens — **MAJUSCULES obligatoires**

```spl
error AND login
error OR warning
NOT debug
(error OR fail) AND login
```

> ⚠️ **Piège :** `AND` est implicite entre deux termes. `error login` = `error AND login`.
> `and` en minuscule est cherché comme un **terme littéral**, pas comme opérateur !

**Précédence :** `NOT` > `AND` > `OR` — utilisez des parenthèses pour lever toute ambiguïté.

### Wildcards

| Syntaxe | Résultat |
|---------|----------|
| `fail*` | fail, failed, failure... |
| `*error*` | contient "error" n'importe où |
| `192.168.*` | toute IP 192.168.x.x |
| `user_?` | user_a, user_1 (exactement 1 caractère) |

---

## 5. Champs par défaut

Ces champs sont **présents sur chaque événement**, créés automatiquement par Splunk.

| Champ | Description | Exemple |
|-------|-------------|---------|
| `_time` | Timestamp de l'événement (Unix epoch) | `1711234567` |
| `_raw` | Données brutes de l'événement | Texte original du log |
| `host` | Hôte source des données | `webserver-01` |
| `source` | Chemin du fichier ou URI source | `/var/log/syslog` |
| `sourcetype` | Format/type des données | `access_combined`, `syslog` |
| `index` | Index où l'événement est stocké | `main`, `web`, `security` |
| `splunk_server` | Serveur Splunk ayant indexé l'événement | `splunk-idx-01` |

> **Note :** Les champs préfixés `_` (`_time`, `_raw`) sont des **champs internes**. Ils existent pour chaque événement mais ne comptent pas dans les champs "selected" ou "interesting".

---

## 6. Search modes

| Mode | Comportement | Quand l'utiliser |
|------|-------------|-----------------|
| **Fast mode** | Moins de champs extraits, plus rapide. Désactive l'extraction de certains champs. | Recherches larges, monitoring rapide |
| **Smart mode** | Par défaut. Equilibre vitesse et richesse des champs. | Usage quotidien |
| **Verbose mode** | Extrait tous les champs possibles. Plus lent mais complet. | Investigation, débogage |

---

## 7. Commandes de filtrage & transformation

| Commande | Syntaxe | Rôle |
|----------|---------|------|
| `search` | `\| search status=200` | Filtre supplémentaire après un pipe |
| `where` | `\| where count > 100` | Filtre avec expressions (comparaisons, fonctions) |
| `fields` | `\| fields host, status, uri` | Conserve ou supprime des champs |
| `rename` | `\| rename clientip AS "IP Source"` | Renomme un champ |
| `table` | `\| table _time, host, status` | Affiche une table avec les champs spécifiés |
| `head` | `\| head 20` | N premiers résultats (défaut : 10) |
| `tail` | `\| tail 10` | N derniers résultats |
| `sort` | `\| sort -count, +host` | Tri (`-` décroissant, `+` croissant) |
| `dedup` | `\| dedup host` | Supprime les doublons sur un champ |
| `eval` | `\| eval total=price*qty` | Crée ou modifie un champ avec une expression |
| `rex` | `\| rex "(?P<user>\w+)"` | Extraction par regex (groupe nommé) |

---

## 8. Commande `fields`

### Inclusion (conserver des champs)

```spl
| fields host status uri
```
→ Garde **seulement** ces 3 champs, supprime tous les autres.

### Exclusion (supprimer des champs)

```spl
| fields - _raw splunk_server
```
→ Supprime ces champs, **garde tous les autres**.

> ⚠️ **Piège examen :** Le `-` devant le nom du champ signifie **exclusion**. Sans `-`, c'est une **inclusion**. Confondre les deux est une erreur classique.

---

## 9. Commande `eval`

### Calculs numériques

```spl
| eval total = price * quantity
| eval margin = round((revenue - cost) / revenue * 100, 2)
```

### Conditions

```spl
| eval severity = if(status >= 500, "critical", "normal")
| eval label = case(status==200,"OK", status==404,"Not Found", 1==1,"Other")
```

> `1==1` dans `case()` est l'équivalent d'un `else` — toujours vrai, donc capte tout le reste.

### Chaînes de caractères

```spl
| eval fullname = first . " " . last       ← concaténation avec "."
| eval domain = lower(host)
| eval short_url = substr(uri, 1, 20)
```

### Gestion du temps

```spl
| eval readable_time = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval ts = strptime("2024-01-15", "%Y-%m-%d")
```

> **À retenir :** `strftime` = timestamp → chaîne lisible. `strptime` = chaîne → timestamp. Ce sens est souvent demandé à l'examen.

---

## 10. Commande `rex`

### Extraction d'un groupe nommé depuis `_raw`

```spl
| rex "user=(?P<username>\w+)"
```
→ Crée le champ `username`.

### Mode `field=` — appliquer sur un champ spécifique

```spl
| rex field=uri "\/api\/(?P<endpoint>[^\/]+)"
```

### Mode `sed` — remplacer ou masquer

```spl
| rex mode=sed "s/\d{4}-\d{4}-\d{4}-\d{4}/XXXX-XXXX-XXXX-XXXX/g"
```

> **Syntaxe :** `(?P<nom_champ>pattern)` = convention Python pour les groupes nommés. C'est la syntaxe standard Splunk pour créer un champ via `rex`.

---

## 11. Commande `stats`

### Comptage

```spl
| stats count
| stats count by host, status
```

### Fonctions numériques

```spl
| stats sum(bytes) AS total_bytes by host
| stats avg(response_time) by uri
| stats min(price) max(price) by category
| stats dc(user) AS unique_users          ← distinct count
| stats values(status) by host            ← liste des valeurs uniques
| stats list(status) by host              ← toutes les valeurs (doublons inclus)
| stats first(status) last(status) by session
```

### Référence des fonctions

| Fonction | Description |
|----------|-------------|
| `count` | Nombre d'événements |
| `dc(champ)` | Nombre de valeurs **distinctes** |
| `sum(champ)` | Somme |
| `avg(champ)` | Moyenne |
| `min(champ)` / `max(champ)` | Valeur min / max |
| `values(champ)` | Liste des valeurs uniques (champ multivalué) |
| `list(champ)` | Toutes les valeurs avec doublons |
| `earliest(champ)` / `latest(champ)` | Valeur la plus ancienne / récente |
| `perc95(champ)` | 95e percentile |
| `range(champ)` | Écart entre max et min |

> ⚠️ **Après une commande `stats`**, l'onglet "Events" n'est plus disponible — les résultats sont agrégés.

---

## 12. Commandes `chart` vs `timechart`

| | `chart` | `timechart` |
|--|---------|------------|
| **Axe X** | N'importe quel champ | Toujours `_time` |
| **Syntaxe** | `\| chart count over host by status` | `\| timechart count by status` |
| **Option** | — | `span=1h` pour définir le bucket temps |

```spl
| chart count over host by status
→ X = host, colonnes = valeurs de status

| timechart span=1h avg(bytes) by host
→ X = temps (buckets d'1h), séries = valeurs de host
```

> Si `span` n'est pas précisé dans `timechart`, Splunk choisit automatiquement selon la plage de temps sélectionnée.

---

## 13. Commandes `top` et `rare`

```spl
| top clientip               ← 10 IPs les plus fréquentes (défaut : 10)
| top limit=5 status by host ← top 5 status par host
| top limit=0 uri            ← toutes les valeurs (pas de limite)

| rare clientip              ← IPs les moins fréquentes
```

**Sortie automatique de `top` et `rare` :**

| Colonne | Description |
|---------|-------------|
| valeur du champ | La valeur observée |
| `count` | Nombre d'occurrences |
| `percent` | Pourcentage du total |

> **Options utiles :** `showperc=f` pour masquer le pourcentage. `countfield=nb` pour renommer la colonne count.

---

## 14. Types de champs

| Type | Description | Exemple |
|------|-------------|---------|
| **Default fields** | Présents sur tous les événements, créés par Splunk | `_time`, `host`, `source`, `sourcetype`, `index` |
| **Extracted fields** | Extraits du texte brut au moment de la recherche | `status`, `clientip` dans `access_combined` |
| **Indexed fields** | Extraits et stockés à l'indexation (rare, coûteux) | Champs configurés dans `transforms.conf` |
| **Calculated fields** | Calculés via `eval` lors de la recherche | `eval total=price*qty` |
| **Lookup fields** | Enrichis depuis une table externe (CSV, KV Store) | `country` depuis un IP lookup |

---

## 15. Selected vs Interesting fields

| | Selected fields | Interesting fields |
|--|----------------|-------------------|
| **Définition** | Toujours affichés dans la sidebar | Champs présents dans ≥ 20% des événements |
| **Par défaut** | `host`, `source`, `sourcetype` | Variable selon les données |
| **Ajout** | Clic "Add to selected fields" | Automatique si seuil atteint |

> ⚠️ Un champ présent dans **moins de 20%** des événements n'apparaît pas automatiquement comme "interesting". Pour le trouver : cliquer sur "All fields" ou utiliser `| fieldsummary`.

---

## 16. Tags & Event Types

### Tags

Les tags sont des **étiquettes** attribuées à des valeurs de champs. Ils permettent des recherches sémantiques transversales.

```spl
← Si le tag "web_server" est associé à host=srv-01 :
tag=web_server
→ Trouve tous les événements de srv-01
```

### Event Types

Un event type est une **recherche sauvegardée sous un nom**, qui crée un champ `eventtype` sur les événements correspondants.

```spl
← Event type "http_errors" défini par : index=web status>=400

eventtype=http_errors
→ Trouve tous les événements avec status >= 400
```

> **Points clés :**
> - Un event type ajoute le champ `eventtype` aux événements correspondants.
> - Un événement peut avoir **plusieurs eventtypes** simultanément (champ multivalué).
> - Les tags s'appliquent à une **paire champ=valeur** spécifique.

---

## 17. Field extraction — méthodes

| Méthode | Description | Quand l'utiliser |
|---------|-------------|-----------------|
| **IFX** (Interactive Field Extractor) | Interface graphique, clic sur des exemples dans les événements | Extraction simple, sourcetype connu |
| **Regex (ERE)** | Expression régulière avec groupe nommé via `rex` ou Settings | Patterns complexes ou variables |
| **Delimiters** | Découpage par séparateur (CSV, pipe, tab) | Données structurées et régulières |

---

## 18. Visualisations

| Visualisation | Prérequis des données | Cas d'usage |
|---------------|----------------------|-------------|
| **Bar / Column chart** | `stats`/`chart` avec un champ catégoriel | Comparaisons entre catégories |
| **Line chart** | `timechart` ou `chart` avec axe continu | Tendances dans le temps |
| **Area chart** | Idem line chart | Volume cumulé dans le temps |
| **Pie / Donut** | `stats count by champ` (peu de valeurs) | Répartition en % (< 10 catégories) |
| **Scatter plot** | 2 champs numériques minimum | Corrélation entre métriques |
| **Bubble chart** | 3 champs numériques (X, Y, taille) | Corrélation + volume |
| **Single value** | 1 seul chiffre dans les résultats | KPI, compteur, pourcentage |
| **Gauge** | 1 seul chiffre + plages configurées | Seuils d'alerte visuels |
| **Choropleth map** | Champ géographique (`country`, `region`, `zip`) | Distribution géographique |
| **Cluster map** | Champs `latitude` + `longitude` | Points GPS |

> ⚠️ L'onglet **Visualization** n'est disponible **qu'après une commande de transformation** (`stats`, `chart`, `timechart`, `top`, `rare`...). Sur des événements bruts, seul l'onglet Events est actif.

---

## 19. Dashboards

### Types de panels

- Search panel (SPL inline)
- Visualization panel (graphique)
- Single value panel (KPI)
- Table panel (données tabulaires)
- Text / HTML panel (contenu statique)

### Permissions

| Niveau | Accès |
|--------|-------|
| **Private** | Auteur seul |
| **Shared in App** | Utilisateurs de l'app |
| **Shared globally** | Tous les utilisateurs Splunk |

> **Fonctionnalités avancées :** Les **drilldown** permettent de déclencher une recherche ou d'ouvrir une page en cliquant sur un graphique. Les **input tokens** (time pickers, dropdowns) rendent un dashboard interactif.

---

## 20. Rapports (Reports)

Un rapport = une **recherche sauvegardée** pouvant être planifiée, partagée et intégrée dans un dashboard.

| Paramètre | Description |
|-----------|-------------|
| **Schedule** | Fréquence d'exécution : toutes les heures, jours, CRON custom (ex. `0 8 * * 1-5`) |
| **Time range** | Fenêtre de recherche utilisée lors de l'exécution planifiée |
| **Acceleration** | Précalcule les résultats pour accélérer les recherches (Report Acceleration) |
| **Permissions** | Private, App, ou Global — contrôle qui peut voir/modifier |
| **Embed** | Permet d'intégrer le rapport dans un dashboard ou une page externe |

---

## 21. Alertes

### Types d'alertes

| Type | Description |
|------|-------------|
| **Scheduled** | Exécutée selon un planning (cron). Recommandée en production. |
| **Real-time** | Continue, déclenche dès qu'un résultat arrive. Consomme plus de ressources. |

### Conditions de déclenchement

| Condition | Description |
|-----------|-------------|
| **Per-result** | Une alerte par résultat trouvé |
| **Number of results** | Si `count > N` |
| **Number of hosts** | Si `dc(host) > N` |
| **Number of sources** | Si `dc(source) > N` |
| **Custom condition** | Expression SPL personnalisée |

### Actions disponibles

- Send email
- Run a script
- Add to triggered alerts
- Webhook
- Log event
- Output to lookup

### Throttling

Le **throttling** évite la tempête d'alertes en supprimant les déclenchements répétés pendant une période configurée.

Exemple : même si la condition est vraie toutes les minutes, l'alerte n'est envoyée qu'une fois par heure. On peut également throttler **par champ** (ex. ne pas alerter deux fois pour le même `host` dans la fenêtre).

> ⚠️ Les alertes **real-time** consomment beaucoup de ressources. Pour la production, les alertes **scheduled** sont préférées.

---

## 22. Lookups

Un lookup permet d'**enrichir des événements** avec des données d'une table externe (CSV ou KV Store) en faisant correspondre un champ commun.

### Commandes principales

| Commande | Rôle |
|----------|------|
| `lookup` | Enrichit les événements en ajoutant des champs depuis la table de lookup |
| `inputlookup` | Charge directement la table de lookup comme source de données (remplace l'index) |
| `outputlookup` | Écrit les résultats d'une recherche dans un fichier de lookup (CSV) |

### Syntaxe

```spl
| Lookup simple
index=web | lookup geo_lookup clientip
→ ajoute country, region, city si clientip correspond

| Lookup avec renommage du champ clé
index=web | lookup user_lookup username AS user
→ cherche le champ local "user" dans la colonne "username" du lookup

| Lookup avec OUTPUT explicite
| lookup product_lookup product_id OUTPUT product_name, category

| Lookup avec OUTPUTNEW (ne surcharge pas les champs existants)
| lookup ip_lookup clientip OUTPUTNEW country

| Charger une table lookup directement
| inputlookup users.csv
| inputlookup users.csv where department="IT"

| Sauvegarder les résultats dans un lookup
index=web | stats count by status | outputlookup daily_stats.csv
```

> ⚠️ **OUTPUT vs OUTPUTNEW :** `OUTPUT` écrase les champs existants. `OUTPUTNEW` n'ajoute le champ que s'il **n'existe pas déjà**. Cette distinction est souvent testée à l'examen.

### Automatic lookups

Un **automatic lookup** (configuré dans Settings → Lookups → Automatic lookups) s'applique automatiquement à tous les événements d'un sourcetype donné, sans écrire `| lookup` dans chaque recherche.

> Si un champ comme `country` apparaît sans `| lookup` dans la recherche, c'est probablement un automatic lookup configuré sur le sourcetype.

### Types de tables de lookup

| Type | Description | Cas d'usage |
|------|-------------|-------------|
| **CSV lookup** | Fichier CSV uploadé dans Splunk | Référentiels statiques : IP → pays, user → département |
| **KV Store lookup** | Base clé-valeur intégrée à Splunk (MongoDB) | Données dynamiques, mises à jour fréquentes |
| **External lookup** | Script Python/bash externe | Enrichissement depuis une API ou BDD externe |
| **Geospatial lookup** | Table de formes géographiques (KMZ) | Cartes choroplèthes avec zones personnalisées |

---

*Bonne révision — Good luck pour l'examen SPLK-1001 !*
