# Fiches de révision — Splunk Core User (SPLK-1001) & Power User (SPLK-1002)

---

## Sommaire

### 🟦 Core User (SPLK-1001)

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
23. [Working with Time](#23-working-with-time)

### 🟧 Power User (SPLK-1002) — sujets supplémentaires

24. [Sous-recherches](#24-sous-recherches)
25. [Commandes de combinaison — `append`, `appendcols`, `join`](#25-commandes-de-combinaison--append-appendcols-join)
26. [Commande `transaction`](#26-commande-transaction)
27. [Commandes `eventstats` et `streamstats`](#27-commandes-eventstats-et-streamstats)
28. [Commandes `xyseries` et `untable`](#28-commandes-xyseries-et-untable)
29. [Commande `map`](#29-commande-map)
30. [Search Macros](#30-search-macros)
31. [Field Aliases & Calculated Fields](#31-field-aliases--calculated-fields)
32. [Workflow Actions](#32-workflow-actions)
33. [Lookups avancés](#33-lookups-avancés)
34. [Data Models, Pivot & `tstats`](#34-data-models-pivot--tstats)

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

## 23. Working with Time

### Unités de temps — référence complète

| Abréviation | Unité |
|-------------|-------|
| `s` | secondes |
| `m` | minutes |
| `h` | heures |
| `d` | jours |
| `w` | semaines |
| `mon` | mois |
| `q` | trimestres |
| `y` | années |

---

### Time range picker — `earliest` et `latest`

Les paramètres `earliest` et `latest` peuvent être passés directement dans la recherche SPL pour surcharger le time range picker.

```spl
earliest=-24h latest=now
earliest=-7d@d latest=@d
earliest=-1mon latest=-1d@d
```

---

### Le modificateur `@` — snap to time

Le `@` **arrondit** (snap) au début de l'unité de temps spécifiée.

| Expression | Signification |
|------------|---------------|
| `-7d@d` | Il y a 7 jours, arrondi au **début du jour** (00:00:00) |
| `-1h@h` | Il y a 1 heure, arrondi au **début de l'heure** |
| `@w0` | Début de semaine = **dimanche** |
| `@w1` | Début de semaine = **lundi** |
| `@mon` | Début du mois courant |
| `@y` | Début de l'année courante |
| `-1y@y` | Début de l'année **précédente** |

> ⚠️ **Piège examen :** `@w0` = dimanche, `@w1` = lundi. La semaine commence le dimanche dans Splunk (convention américaine).

---

### Temps relatif — exemples courants

```spl
earliest=-30m             ← 30 dernières minutes
earliest=-2h@h            ← 2h en arrière, arrondi au début de l'heure
earliest=-1d@d latest=@d  ← hier, du début à la fin de journée
earliest=@w1 latest=now   ← depuis le lundi de la semaine courante
earliest=-1mon@mon latest=@mon  ← le mois précédent complet
```

---

### Temps absolu

Le format pour spécifier un temps absolu dans SPL est : `MM/DD/YYYY:HH:MM:SS`

```spl
earliest="01/15/2024:00:00:00" latest="01/15/2024:23:59:59"
```

---

### Fonctions de temps dans `eval`

#### `now()` — timestamp actuel

```spl
| eval current_time = now()
| eval age_seconds = now() - _time
```

#### `time()` — timestamp de l'événement (identique à `_time`)

```spl
| eval event_epoch = time()
```

#### `strftime()` — timestamp → chaîne lisible

```spl
| eval readable = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval date_only = strftime(_time, "%d/%m/%Y")
| eval hour = strftime(_time, "%H")
```

**Formats courants :**

| Format | Résultat |
|--------|----------|
| `%Y` | Année sur 4 chiffres (2024) |
| `%m` | Mois sur 2 chiffres (01–12) |
| `%d` | Jour sur 2 chiffres (01–31) |
| `%H` | Heure sur 2 chiffres, 24h (00–23) |
| `%M` | Minutes (00–59) |
| `%S` | Secondes (00–59) |
| `%A` | Nom du jour (Monday, Tuesday...) |
| `%B` | Nom du mois (January, February...) |

#### `strptime()` — chaîne → timestamp (epoch)

```spl
| eval ts = strptime("2024-01-15 08:30:00", "%Y-%m-%d %H:%M:%S")
```

> **Mémo :** `strftime` = **f**ormat → lisible. `strptime` = **p**arse → epoch.

#### `relative_time()` — calculer un temps relatif par rapport à un epoch

```spl
| eval start_of_day = relative_time(now(), "@d")
| eval last_monday  = relative_time(now(), "-7d@w1")
| eval hour_ago     = relative_time(now(), "-1h")
```

> `relative_time(X, Y)` prend un **epoch X** et un **modificateur de temps Y** (même syntaxe que `earliest`/`latest`), et retourne un nouvel epoch.

---

### Commande `bin` / `bucket` — regrouper par tranche de temps

`bin` et `bucket` sont des **synonymes**. Ils permettent de regrouper des valeurs numériques ou temporelles en tranches.

```spl
| bin _time span=1h
| stats count by _time

| bin _time span=30m
| stats avg(response_time) by _time

| bin response_time span=100    ← fonctionne aussi sur des valeurs numériques
| stats count by response_time
```

> **Différence avec `timechart` :** `timechart` fait le `bin` + `stats` en une seule commande. Avec `bin`, on contrôle chaque étape séparément, ce qui offre plus de flexibilité.

---

### Résumé des pièges examen — Working with Time

| Piège | À retenir |
|-------|-----------|
| `@w0` vs `@w1` | `@w0` = dimanche, `@w1` = lundi |
| `strftime` vs `strptime` | `strftime` → lisible, `strptime` → epoch |
| `now()` vs `_time` | `now()` = moment de la recherche, `_time` = timestamp de l'événement |
| `relative_time()` | Retourne un epoch, pas une chaîne — à combiner avec `strftime` pour l'afficher |
| `bin` vs `timechart` | `timechart` = bin + stats intégré. `bin` = étape manuelle, plus flexible |
| `earliest=-7d@d` | Le `@d` arrondit au début du jour, **pas** il y a 7 jours à l'heure exacte |

---

---

# 🟧 Power User (SPLK-1002) — sujets supplémentaires

> Les sections suivantes couvrent les sujets **spécifiques au Power User** qui ne sont pas au programme du Core User. Tout ce qui précède reste valable et est considéré comme acquis.

---

## 24. Sous-recherches

Une **sous-recherche** est une recherche imbriquée dans une autre, entre crochets `[ ]`. Elle est exécutée en premier, et son résultat est injecté dans la recherche principale.

```spl
index=web [search index=threat_intel | fields src_ip | rename src_ip AS clientip]
```

→ Récupère d'abord les IPs malveillantes depuis `threat_intel`, puis filtre les événements web correspondants.

### Syntaxe générale

```spl
recherche_principale [sous-recherche]
```

### Exemple — trouver les utilisateurs ayant eu une erreur puis un succès

```spl
index=auth action=success
  [search index=auth action=failure | stats count by user | fields user]
```

### Contraintes importantes

| Contrainte | Valeur par défaut |
|------------|------------------|
| Nombre max de résultats retournés | 10 000 événements |
| Temps d'exécution max | 60 secondes |
| Résultat injecté | Comme filtre `OR` sur les valeurs du premier champ |

> ⚠️ **Piège examen :** La sous-recherche retourne par défaut les valeurs du **premier champ** sous forme de filtre. Si on veut un champ précis, utiliser `| return` ou `| fields` pour le contrôler.

### Commande `return`

`return` contrôle précisément ce que la sous-recherche renvoie à la recherche principale.

```spl
index=web [search index=auth action=failure | return 5 user]
```
→ Retourne les 5 premières valeurs du champ `user` comme filtre `user=val1 OR user=val2...`

```spl
| return 10 $user    ← le $ retourne la paire champ=valeur complète
```

---

## 25. Commandes de combinaison — `append`, `appendcols`, `join`

### `append` — empiler des résultats

Ajoute les résultats d'une sous-recherche **en dessous** des résultats courants (ajout de lignes).

```spl
index=web status=200 | stats count AS successes
| append [search index=web status>=400 | stats count AS errors]
```

> Les deux jeux de résultats sont **empilés verticalement**. Les colonnes non communes restent vides.

### `appendcols` — ajouter des colonnes

Ajoute les résultats d'une sous-recherche **à droite** des résultats courants (ajout de colonnes).

```spl
index=web | stats count by host
| appendcols [search index=web status=200 | stats count AS ok_count by host]
```

> Les lignes sont appariées **par position** (ligne 1 avec ligne 1...), pas par valeur de champ. Attention à l'ordre !

### `join` — jointure sur un champ commun

Fonctionne comme un `JOIN` SQL. Combine deux jeux de résultats sur un champ clé.

```spl
index=web | join clientip [search index=threat_intel | fields src_ip, threat_level | rename src_ip AS clientip]
```

| Option | Description |
|--------|-------------|
| `type=inner` (défaut) | Garde seulement les lignes ayant une correspondance |
| `type=left` | Garde toutes les lignes de gauche, même sans correspondance |
| `type=outer` | Equivalent de `left` dans Splunk |
| `max=0` | Pas de limite sur le nombre de correspondances (défaut : 1) |

> ⚠️ `join` peut être coûteux en performance. Préférer `lookup` quand c'est possible.

---

## 26. Commande `transaction`

`transaction` regroupe des événements liés en une **session** ou **transaction**, selon un champ commun et/ou des conditions de début/fin.

```spl
index=web | transaction clientip maxspan=30m maxpause=5m
```

### Paramètres principaux

| Paramètre | Description | Exemple |
|-----------|-------------|---------|
| `maxspan` | Durée maximale totale de la transaction | `maxspan=1h` |
| `maxpause` | Délai max entre deux événements consécutifs | `maxpause=5m` |
| `startswith` | Condition de début de transaction | `startswith=eval(action=="login")` |
| `endswith` | Condition de fin de transaction | `endswith=eval(action=="logout")` |
| `maxevents` | Nombre max d'événements dans une transaction | `maxevents=10` |

### Champs créés automatiquement

| Champ | Description |
|-------|-------------|
| `duration` | Durée en secondes entre le premier et le dernier événement |
| `eventcount` | Nombre d'événements dans la transaction |
| `_raw` | Concaténation de tous les événements bruts |

```spl
index=web
| transaction sessionid maxspan=30m
| where duration > 60
| stats avg(duration) by clientip
```

> ⚠️ **`transaction` vs `stats` :** `stats` est beaucoup plus rapide et scalable. N'utiliser `transaction` que si on a besoin de `duration`, `eventcount`, ou de conditions `startswith`/`endswith`.

---

## 27. Commandes `eventstats` et `streamstats`

### `eventstats` — stats sans transformer les événements

`eventstats` calcule des agrégations comme `stats`, mais **ajoute les résultats comme nouveaux champs sur chaque événement** au lieu de remplacer les événements.

```spl
index=web
| eventstats avg(bytes) AS avg_bytes
| where bytes > avg_bytes * 2
```

→ Chaque événement conserve ses champs originaux **plus** le nouveau champ `avg_bytes`.

```spl
| eventstats count by host        ← count par host ajouté sur chaque événement
| eventstats sum(bytes) AS total  ← total global ajouté sur chaque ligne
```

> **Différence clé :**
> - `stats` → remplace les événements par un tableau agrégé
> - `eventstats` → enrichit chaque événement avec des valeurs agrégées, les événements restent intacts

### `streamstats` — stats cumulatives ligne par ligne

`streamstats` calcule des agrégations de manière **progressive**, en traitant les événements dans l'ordre (du plus récent au plus ancien par défaut).

```spl
index=web
| sort _time
| streamstats count AS event_number
| streamstats sum(bytes) AS cumulative_bytes by clientip
```

| Paramètre | Description |
|-----------|-------------|
| `window=N` | Calcule sur les N derniers événements (fenêtre glissante) |
| `current=f` | Exclut l'événement courant du calcul |
| `global=f` | Calcul par groupe (avec `by`) sans réinitialiser globalement |
| `reset_on_change=t` | Remet à zéro quand la valeur du champ `by` change |

```spl
| streamstats window=5 avg(response_time) AS moving_avg
```
→ Moyenne mobile sur les 5 derniers événements.

---

## 28. Commandes `xyseries` et `untable`

Ces deux commandes servent à **pivoter** les données — passer d'un format long à un format large, ou inversement.

### `xyseries` — format long → large (pivot)

```spl
index=web | stats count by _time, status
| xyseries _time status count
```

→ Crée une colonne par valeur de `status` (200, 404, 500...), avec `_time` comme ligne.

**Syntaxe :** `| xyseries <colonne_X> <colonne_séries> <valeur>`

### `untable` — format large → long (dépivot)

`untable` fait l'inverse de `xyseries` : transforme un tableau large en format long.

```spl
| untable _time status count
```

**Syntaxe :** `| untable <colonne_X> <nom_colonne_séries> <nom_colonne_valeur>`

---

## 29. Commande `map`

`map` exécute une recherche pour **chaque ligne** du jeu de résultats courant, en substituant les valeurs de champs comme variables.

```spl
index=web | top limit=5 clientip
| map search="search index=web clientip=$clientip$ | stats count by uri | head 3"
```

→ Pour chacune des 5 IPs, lance une recherche SPL et agrège les résultats.

> ⚠️ `map` est puissant mais **très coûteux** — il lance autant de recherches qu'il y a de lignes. Limiter avec `| head` avant `map`. Défaut : max 10 recherches (`maxsearches=10`).

---

## 30. Search Macros

Une **search macro** est un raccourci réutilisable pour un fragment de SPL, défini dans Settings → Advanced Search → Search macros.

### Macro sans argument

```spl
← Définition de la macro "web_errors" :
index=web status>=400

← Utilisation dans une recherche :
`web_errors` | stats count by status
```

> Les macros s'appellent avec des **backticks** `` ` ``.

### Macro avec arguments

```spl
← Définition de "filter_status(1)" avec l'argument "status_code" :
index=web status=$status_code$

← Utilisation :
`filter_status(404)` | stats count by clientip
`filter_status(500)` | timechart count
```

### Macro avec plusieurs arguments

```spl
← Définition de "index_filter(2)" avec "idx" et "stype" :
index=$idx$ sourcetype=$stype$

← Utilisation :
`index_filter(web, access_combined)` | head 20
```

> **Validation :** On peut ajouter une **validation d'argument** (expression + message d'erreur) dans la définition de la macro pour éviter les mauvaises valeurs.

### Macro avec `iseval`

```spl
← Définition de "last_N_hours(1)" :
earliest=-$hours$h

← Utilisation :
`last_N_hours(6)` index=web | stats count
```

---

## 31. Field Aliases & Calculated Fields

### Field Aliases

Un **field alias** permet de donner un **nom alternatif** à un champ existant, sans modifier les données. Utile pour normaliser des champs de sourcetypes différents.

Exemple : `src_ip` dans un sourcetype et `clientip` dans un autre → créer un alias `source_address` sur les deux.

- Configuré dans : **Settings → Fields → Field aliases**
- S'applique à un sourcetype, une source, ou un host
- Le champ original reste accessible, l'alias est un nom supplémentaire

### Calculated Fields

Un **calculated field** est un champ `eval` **sauvegardé** qui s'applique automatiquement à chaque recherche sur un sourcetype donné, sans avoir à réécrire l'`eval`.

```spl
← Plutôt que d'écrire à chaque fois :
| eval response_kb = bytes / 1024

← On définit "response_kb" comme calculated field sur le sourcetype access_combined
← Il apparaît automatiquement dans tous les résultats
```

- Configuré dans : **Settings → Fields → Calculated fields**
- Basé sur une expression `eval`
- Appliqué automatiquement sur le sourcetype/source/host ciblé

> **Ordre d'application :** Field aliases sont appliqués **avant** les calculated fields.

---

## 32. Workflow Actions

Les **workflow actions** ajoutent des actions contextuelles sur les champs d'un événement (clic droit ou menu sur un champ dans les résultats).

### Types de workflow actions

| Type | Description | Exemple |
|------|-------------|---------|
| **GET / POST (lien)** | Ouvre une URL externe avec la valeur du champ | Rechercher une IP dans VirusTotal |
| **Search** | Lance une nouvelle recherche Splunk | Voir tous les events de ce `clientip` |

### Configuration

- **Settings → Fields → Workflow actions**
- On définit : le label, les champs déclencheurs, le type (lien ou recherche), l'URL ou la SPL

```
Label : "Lookup IP on VirusTotal"
Apply to fields : clientip
Type : link
URL : https://www.virustotal.com/gui/ip-address/$clientip$
```

---

## 33. Lookups avancés

### Lookups avec wildcards

Permet de faire correspondre des patterns plutôt que des valeurs exactes. Utile pour les plages d'URL, les catégories, etc.

```spl
← Dans la définition du lookup (transforms.conf) : WILDCARD(field) activé
← Exemple : table avec url_pattern=*/admin/* → category=admin

index=web | lookup url_categories url AS uri OUTPUT category
```

### Lookups avec CIDR (plages IP)

Permet de faire correspondre une IP à une plage réseau définie dans le lookup.

```spl
← Table lookup avec une colonne "network" contenant des CIDR : 192.168.0.0/16

index=web | lookup network_lookup clientip AS src CIDR(network) OUTPUT zone
```

### `inputlookup` avec `where` et `append`

```spl
| inputlookup users.csv where role="admin"

← Combiner deux lookups :
| inputlookup table_a.csv
| append [| inputlookup table_b.csv]
```

### `outputlookup` — options avancées

```spl
| outputlookup createinapp=true users_export.csv   ← crée dans l'app courante
| outputlookup append=true daily_log.csv            ← ajoute sans écraser
| outputlookup max=0 full_export.csv                ← sans limite de lignes
```

---

## 34. Data Models, Pivot & `tstats`

### Data Models

Un **data model** est une représentation hiérarchique et structurée de données, basée sur des contraintes (champs obligatoires, types). Il sert de fondation au **Pivot** et à `tstats`.

- Basé sur le **CIM (Common Information Model)** — standard Splunk pour normaliser les données
- Organisé en **datasets** (nœuds) avec des contraintes sur les champs
- Peut être **accéléré** (précalcul des données en arrière-plan → tsidx files)

### Pivot

**Pivot** est une interface graphique pour créer des rapports et visualisations **sans écrire de SPL**, à partir d'un data model.

- Accessible depuis : **Settings → Data Models → Pivot**
- Equivalent visuel de `stats`/`chart` mais piloté par le data model
- Génère automatiquement la SPL sous-jacente

### `tstats` — recherches ultra-rapides

`tstats` interroge directement les **index accélérés** (tsidx) d'un data model, ce qui est **beaucoup plus rapide** que `stats` sur les données brutes.

```spl
| tstats count FROM datamodel=Web.Web BY Web.uri, Web.status
| rename Web.uri AS uri, Web.status AS status
```

```spl
| tstats summariesonly=t count, sum(Web.bytes) AS total_bytes
  FROM datamodel=Web.Web
  WHERE Web.status=200
  BY _time span=1h, Web.clientip
```

| Paramètre | Description |
|-----------|-------------|
| `summariesonly=t` | N'utilise que les données accélérées (plus rapide, pas de backfill) |
| `summariesonly=f` | Inclut aussi les données non encore accélérées (défaut) |
| `FROM datamodel=X.Y` | Spécifie le dataset (`X` = data model, `Y` = dataset racine ou enfant) |
| `BY _time span=1h` | Groupement temporel comme `timechart` |

> **À retenir :** `tstats` est à `stats` ce que les index accélérés sont aux index normaux — même logique, performances radicalement différentes sur de gros volumes.

### CIM — Common Information Model

Le CIM définit des **noms de champs normalisés** communs à tous les sourcetypes. Exemples :

| Domaine CIM | Champs normalisés |
|-------------|------------------|
| **Web** | `src`, `dest`, `uri`, `status`, `bytes`, `method` |
| **Authentication** | `user`, `src`, `dest`, `action` (success/failure) |
| **Network Traffic** | `src_ip`, `dest_ip`, `src_port`, `dest_port`, `transport` |
| **Endpoint** | `host`, `user`, `process`, `process_name` |

---

*Bonne révision — Good luck pour les examens SPLK-1001 & SPLK-1002 !*
