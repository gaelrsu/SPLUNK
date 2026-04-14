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

searchpeer : équivalent à indexer

Splunk diag : créé un tar gz avec info sur conf (attention ne prend pas les logs / events) 
montrera l'index mais pas le contenu 

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
































