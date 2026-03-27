# Using Fields

## Architecture Splunk — les composants essentiels
Forwarder (collecte)
Collecte et transmet les données. Universal Forwarder = léger, recommandé. Heavy Forwarder = parsing local possible.
Indexer (stockage)
Reçoit, parse, indexe les données dans des buckets. Effectue les recherches sur ses index.
Search Head (UI)
Interface web pour les utilisateurs. Distribue les recherches aux indexers et agrège les résultats.

## Flux de données — de la source à l'index
1. Input — Splunk consomme des données : fichiers, syslog, scripts, API...
2. Parsing — Découpe en événements, extrait le timestamp, identifie le sourcetype
3. Indexing — Compression, écriture dans les buckets (hot → warm → cold → frozen)
4. Searching — L'utilisateur interroge via SPL, Splunk retourne les événements

## Recherche SPL
```
index=web sourcetype=access_combined
  status=404 uri="/login*"
| stats count by clientip
| sort -count
| head 10
```
Partie	Rôle
index=web sourcetype=...	Critères de recherche (filtrage sur les index/champs)
status=404	Filtre sur un champ extrait ou par défaut
uri="/login*"	Filtre avec wildcard sur un champ
| stats count by	Pipe : passe les résultats à la commande suivante
| sort -count	Tri décroissant (- = décroissant, sans = croissant)
| head 10	Limite à 10 premiers résultats

## Opérateurs booléens et wildcards

Booléens — MAJUSCULES obligatoires
error AND login
error OR warning
NOT debug
(error OR fail) AND login
Wildcards
fail*        → fail, failed, failure
*error*      → contient "error"
192.168.*    → toute IP 192.168.x.x
user_?       → user_a, user_1 (1 char)

## Champs par défaut 

_time	Timestamp de l'événement (Unix epoch)	1711234567
_raw	Données brutes de l'événement	Le texte original du log
host	Hôte source des données	webserver-01
source	Chemin du fichier ou URI source	/var/log/syslog
sourcetype	Format/type des données	access_combined, syslog
index	Index où l'événement est stocké	main, web, security
splunk_server	Serveur Splunk ayant indexé l'événement	splunk-idx-01

## Search modes — impact sur les performances

Fast mode	Moins de champs extraits, plus rapide. Désactive l'extraction de certains champs.	Recherches larges, monitoring rapide
Smart mode	Par défaut. Equilibre vitesse et richesse des champs.	Usage quotidien
Verbose mode	Extrait tous les champs possible. Plus lent mais complet.	Investigation, débogage

## Commandes de filtrage et transformation essentielles

| Commande | Syntaxe | Rôle |
|----------|--------|------|
| search | `| search status=200` | Filtre supplémentaire après un pipe |
| where | `| where count > 100` | Filtre avec expressions (comparaisons, fonctions) |
| fields | `| fields host, status, uri` | Conserve ou supprime des champs |
| rename | `| rename clientip AS "IP Source"` | Renomme un champ |
| table | `| table _time, host, status` | Affiche une table avec les champs spécifiés |
| head | `| head 20` | N premiers résultats (défaut : 10) |
| tail | `| tail 10` | N derniers résultats |
| sort | `| sort -count, +host` | Tri (- décroissant, + croissant) |
| dedup | `| dedup host` | Supprime les doublons sur un champ |
| eval | `| eval total=price*qty` | Crée ou modifie un champ avec une expression |
| rex | `| rex "(?P<user>\w+)"` | Extraction par regex (groupe nommé) |


## fields
— conserver vs supprimer
Conserver des champs (inclusion)
| fields host status uri
→ garde seulement ces 3 champs
Supprimer des champs (exclusion)
| fields - _raw splunk_server
→ supprime ces champs, garde les autres


## eval
— fonctions importantes
| Calculs numériques
| eval total = price * quantity
| eval margin = round((revenue - cost) / revenue * 100, 2)

| Conditions
| eval severity = if(status >= 500, "critical", "normal")
| eval label = case(status==200,"OK", status==404,"Not Found", 1==1,"Other")

| Chaînes de caractères
| eval fullname = first . " " . last
| eval domain = lower(host)
| eval short_url = substr(uri, 1, 20)

| Temps
| eval readable_time = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval ts = strptime("2024-01-15", "%Y-%m-%d")

## rex
— extraction par regex
| Extraction d'un groupe nommé depuis _raw
| rex "user=(?P<username>\w+)"
→ crée le champ "username"

| Mode field= pour appliquer sur un champ spécifique
| rex field=uri "\/api\/(?P<endpoint>[^\/]+)"

| Mode sed pour remplacer/masquer
| rex mode=sed "s/\d{4}-\d{4}-\d{4}-\d{4}/XXXX-XXXX-XXXX-XXXX/g"



