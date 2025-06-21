
## Forwarder
The forwarder is an agent who collects logs and then send it to another Splunk E instance.
The collected data is not parsed but it's the best and most commun way to get data in.

## Indexer
The indexer transforms data into event, store it to the disk and adds it to an index, enabling searchability.
The indexer creates the following files, separating them into directories called bucket :
- Compressed raw dara
- Indexes pointing to raw dara (TSIDX files)
- Metadata files (host, source and source type)
The indexer also performs generic event processing on log data.

## Heavy Forwader
Is a full instance that can index, search and change data as well as forward it.
The difference between Universal Forwarder and Heavy forwarder is that the HF made parsing and indexing. 
When the data comes from a heavy forwarder, inderxers do not parse the data again.
HF are also used to run Splunk add ons that receive data from external sources. Data forwarded are parsed.

## Splunk Search Head
Provides the UI for users to submit searches via SPL. allow users to search anbd query Splunk data.
The Search head sends search requests to a group of indexers.

## deployment Server
A centralized configuration manager. Manage and push configurations to any number of Splunk E instances.
Users interface with the forwarder manager. Deployment client (UFs, HFs) are remotely managed by this instance.
Apps and configirations files are maintained under the "deployment Apss"

## License Master
Centralise license repository, can define license pools an stacks.

## Monitoring console
Is a tool for viewing detailed topology and performance information.

## Splunk Deployer
Is an instance that you use to distribute apps other configuration updates to search head cluster members.

## Cluster Master
Manage and regulate the functioning of indexers so that replicate external data to maintain multiple copiers of the data.
Indexer clusters provide high availability and disaster recovery, have a failover.


