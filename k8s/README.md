# Debezium Connector for SQL Server in k8s

## Introduction
### Dependencies
You need the following on your system, or to be working in an environment with a kubernetes cluster (e.g. an on-premise k8s cluster, or AKS on Azure):
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) - for hosting minikube.  Docker has its own k8s emulator and you can host minikube on other environments, including VMs, (and you can avoid minikube), so maybe you don't need Docker.
- [minikube](https://minikube.sigs.k8s.io/docs/start/) - for testing k8s deployments and objects.  Other environments exist, including ones on the web, e.g. [Play with Kubernetes](https://labs.play-with-k8s.com), but I tested this all on minikube.
- [strimzi](https://strimzi.io/) - the strimzi operator is installed via _kubectl_ once you've installed an OLM (Operator Lifecycle Manager - details shown in Setup section).  Strimzi helps with running kafka and zookeeper clusters in containers.

## Files

### _00-secret.yaml_
- **Needed** to store username and password for database
### _01-role.yaml_
- Needed for assigning a role to access the secrets
### _02-role-binding.yaml_
- **Needed** for role assignment to secrets
### _03-kafka-zookeeper.yaml_
- **Needed** in part - Zookeeper is required for Kafka Connect, Kafka nodes may not be needed (but could be used for internal topics not required for streaming data)
### _04-mssql.yaml_
- **Not needed**, this was for testing and SQL Server won't be hosted in k8s!
### _05-mssql-service.yaml_
- **Not needed**, exposes the (not needed) SQL Server pod (for Kafka Connect)
### _06-connect-cluster.yaml_
- **Needed**, hosts the Debezium libraries for shifting CDC records
### _07-connector.yaml_
- **Needed** as it specifies what to move (database and table specficiations, along with Kafka topics)

## Setup with _minikube_ and _kubectl_
- Start Docker and minikube (`minikube start`)
- Get, and install, the [OLM](https://olm.operatorframework.io) needed by strimzi
  - `curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.25.0/install.sh | bash -s v0.25.0`
- Install the strmzi kafka operator
  - `kubectl create -f https://operatorhub.io/install/strimzi-kafka-operator.yaml`
- Create k8s namespace
  - `kubectl create namespace dbz`
- Install secrets
  - `kubectl apply -n dbz -f 00-secret.yaml`
  - this makes secrets secure in the cluster and accessible (but not human-readable) by other objects
- Protect the secret by assigning a role
  - `kubectl apply -n dbz -f 01-role.yaml`
- Bind the role to the connect service so kafka connect can read it (to access the database being monitored)
  - `kubectl apply -n dbz -f 02-role-binding.yaml`
- Create a kafka and zookeeper deployment; this is a CRD file of type Kafka
  - `kubectl apply -n dbz -f 03-kafka-zookeeper.yaml`
- Create a SQL Server instance and create a database and table that you can monitor
  - `kubectl apply -n dbz -f 04-mssql.yaml`
  - Either open a shell and do some sql or run a file inside the pod...describe one of these (TODO: test the latter)
  - Include some record creation, don't forget to turn on cdc for the new database and table(s)
  - The connector gets an error if the database in the config is:
    - Missing - so create it first
    - Not enable for CDC - so run `EXEC sys.sp_cdc_enable_db; go`
    - Empty - so create a table
    - Empty of CDC-enabled tables - so enable CDC for whichever tables are in there (at least one before the connector is started)
- Create a service for SQL Server (TODO: move this into the SQL file) so your connectors can monitor SQL:
  - `kubectl apply -n dbz -f 05-mssql-service.yaml`
- Create the kafka connect cluster
  - `kubectl apply -n dbz -f 06-connect-cluster.yaml`
  - The output of this is a docker image (see _Build_ section of yaml file).  Make sure the name of the image is the same as the cluster (I think TODO: check what happens if they're different, I think it didn't work when I made them different).
  - The build can take a while.  There's a pod for the cluster ending in _-build_ while it's building; the cluster won't be in a ready state until that's done and the _-[some letter-spaghetti]_ cluster pod is running and ready.
- Create the connector that routes CDC records through to kafka
  - `kubectl apply -n dbz -f 07-connector.yaml`


These steps should create the resources to generate (_SQL Server_), route (_Kafka Connect, Zookeeper_) and receive (_Kafka, Zookeeper_) CDC records from SQL Server via Kafka Connect to Kafka.  In an environment where SQL Server or Kafka are external (i.e. production where Kafka services may be provided by [Azure Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/azure-event-hubs-kafka-overview) and SQL Server is external) some of these resources can be left out (mssql and mssql service) or modified (kafka can be removed from the kafka file, although Zookeeper is still needed for Kafka Connect).

## Monitoring
### Kafka Connect
`kubectl get -n dbz kafkaconnect my-connect-cluster` - show status of Kafka Connect cluster, should look like:
```
NAME                 DESIRED REPLICAS   READY
my-connect-cluster   1                  True
```

(Note the READY indicator set to _TRUE_ , also note this can take a while - check the pod status if it's not ready after ~60 seconds)

`kubectl describe -n dbz kafkaconnect my-current-cluster` to give details of the cluster
### Connector Status
`kubectl describe -n dbz kafkaconnector my-source-connector` should give a running status, including last time it picked up data and the topics it's talking to (topics may only appear once it's sent something to them, they don't always seem to be there):
```
...
Status:
  Conditions:
    Last Transition Time:  2023-08-30T16:05:27.881050211Z
    Status:                True
    Type:                  Ready
  Connector Status:
    Connector:
      State:      RUNNING
      worker_id:  10.244.0.133:8083
    Name:         my-source-connector
    Tasks:
      Id:               0
      State:            RUNNING
      worker_id:        10.244.0.133:8083
    Type:               source
  Observed Generation:  1
  Tasks Max:            1
  Topics:
    sqlserver
    sqlserver.inventory.dbo.era
```
### Kafka
#### List topics
`kubectl exec -n dbz debezium-cluster-kafka-0 -c kafka -i -t -- bin/kafka-topics.sh --bootstrap-server debezium-cluster-kafka-bootstrap:9092 --list`
#### Listen to a topic
`kubectl exec -n dbz debezium-cluster-kafka-0 -c kafka -i -t -- bin/kafka-console-consumer.sh --bootstrap-server debezium-cluster-kafka-bootstrap:9092 --topic sqlserver.inventory.dbo.era --from-beginning`
