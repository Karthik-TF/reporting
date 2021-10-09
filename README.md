# Reporting framework

## Introduction
Reference reporting framework for real-time streaming data and visualization.  

![](docs/images/reporting_architecture.png)

## Installation in Kubernetes cluster

### Prerequisites
 
* MOSIP cluster installed as given [here](https://github.com/mosip/mosip-infra/tree/1.2.0_v3/deployment/v3)
* Elasticsearch and Kibana are already running in the cluster. 
* Postgres is installed with `extended.conf` as extended config. (MOSIP default install has this configured)

###  Install
The `scripts/install.sh` installs the following Helm charts:
  - `mosip/reporting` Helm chart:
    - Debezium Kafka Connect
    - Elasticsearch Kafka Connect 
    - Kafka+Zookeeper _(optional)_
    - Elasticsearch & Kibana _(optional)_
  - `mosip/reporting-init` Helm chart:
    - Debezium-Postgres connectors
    - Elasticsearch-Kafka connectors

All components will be installed in `reporting` namespace of the cluster.

**Install**:
- Inspect `values.yaml` and `values-init.yaml` and configure appropriately.  Leave as is for default installation.
- Run
```sh
cd scripts
./install.sh <kube-config-file>
```

- NOTE: for the db_user use superuser/`postgres` for now, because any other user would require the db_ownership permission, create permission & replication permission. (TODO: solve the problems when using a different user.)
- NOTE: before installing, `reporting-init` debezium configuration, make sure to include all tables under that db beforehand. If one wants to add another table from the same db, it might be harder later on. (TODO: develop some script that adds additional tables under the same db)

## Upload Kibana dashboards
Various dashboards are available in `kibana_dashboards` folder.  Upload all of them with the following script:
```sh
cd scripts
./load_kibana_dashboards.sh
```
The dashboards may also be uploaded manually using Kibana UI.

## Installing addtional connectors
This section is when one wants to install additional connectors that are not present in the reference connectors (or) if one wants to install custom connectors.

- Note: Both the following methods will not add additional tables of existing db to debezium. (Example: it wont add `prereg.otp_transaction`, if other prereg tables have been added before) For this, one will have to edit that db's existing debezium connector manually.

### Method 1:
- Put the new elasticsearch connectors in one folder.
    - Create a configmap with for this folder, using:
    ```
    $ kubectl create configmap <conf-map-name> -n reporting --from-file=<folder-path>
    ```
    - Edit in values-init.yaml, to use the above configmap:
    ```
    es_kafka_connectors:
        existingConfigMap: <conf-map-name>
    ```
- Edit in values-init.yaml, the debezium_connectors for new dbs and tables. Or disable if not required.
    - Can also use a custom debezium connector using the following. (Not recommended)
    - Create a configmap with custom debezium connector:
    ```
    $ kubectl create configmap <conf-map-name> -n reporting --from-file=<path-for-debez-connector>
    ```
    - Edit in values-init.yaml, to use the above configmap:
    ```
    debezium_connectors:
        existingConfigMap: <conf-map-name>
    ```
- Install reporting-init again. (First delete any previously completed instance. This wont affect the cluster/installation)
```
$ helm -n reporting delete reporting-init
$ helm -n reporting install reporting-init mosip/reporting-init -f values-init.yaml
```

### Method 2 (manually):

- Edit the `./sample_connector.api` file, accordingly. And run the following;
```
$ ./run_connect_api.sh sample_connector.api <kube-config-file>
```

## Cleanup/uninstall

- Delete the reporting components
```
$ helm delete reporting-init -n reporting
$ helm delete reporting -n reporting
$ kubectl delete ns reporting
```
- Postgres Cleanup
    - List replication replication slots.
    ```
    postgres=# select * from pg_replication_slots;
    ```
    - Delete each of the above slots.
    ```
    postgres=# select pg_drop_replication_slot('<slot_name>');
    ```
    - Go to each database, and drop publication.
    ```
    postgres=# \c <db>
    postgres=# select * from pg_publication;
    postgres=# drop publication <pub_name>;
    ```
- Kafka Cleanup
    - It is recommended to cleanup all topics related to reporting in kafka, as the data will anyway be there in db/elasticsearch
    - Delete all the relavent topics and the debezium and es kafka connectors' `status`, `offset` and `config` topics.
- Elasticsearch and Kibana Cleanup
    - One can delete the es indices, and delete the dashboards from kibana from the ui, if required.
