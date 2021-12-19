---
layout: post
title: Kafka Connect JDBC Sink Connector (Plain Config and with Schema Registry)
date: 2021-05-04 17:39:00
description: A quick tutorial to setup Kafka Cluster and sink data from the cluster to Oracle Database
---

A quick tutorial to setup Kafka Cluster and sink data from the cluster to Oracle Database

## 1. Basic Kafka Setup

To install `Kafka` on `Mac` use : 
```bash
brew install kafka
```
For brew installation the configs(property files) will be located in -
* Kafka Configs - /usr/local/etc/kafka
* Zookeeper Configs - /usr/local/etc/zookeeper

Start the zookeeper server and kafka server using these commands.
```bash
zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties
 
kafka-server-start /usr/local/etc/kafka/server.properties
```
Now lets create a topic to insert some data into it and use a consumer to read the data back - 
```bash
kafka-topics --zookeeper localhost:2181 --create --replication-factor 1 --partitions 1 --topic test //
Creates a topic named "test"
kafka-console-producer --broker-list localhost:9092 --topic test // This will start a producer to insert
data into the topic "test"
kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning // Will display all
the data inserted in the topic "test"
```

## 2. Kafka Connect JDBC Setup

We will be setting up a JDBC Connector to send data from a kafka topic to Oracle DB.
1. Download the JDBC Connector zip folder (confluentinc-kafka-connect-jdbc-10.2.0.zip) from this link - https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc
2. Extract the zip folder and folder will contain the kafka-connect jar and JDBC drivers such as (MYSQL, Postgre,Oracle etc) in the confluentinc-kafka-connect-jdbc-10.2.0/lib location
3. If the Oracle Driver is not there, manually download it from https://www.oracle.com/database/technologies/
    jdbc-ucp-122-downloads.html and place it in confluentinc-kafka-connect-jdbc-10.2.0/lib location.
4. We will be running connect worker in standalone mode 
5. Creata a connect.offsets file in confluentinc-kafka-connect-jdbc-10.2.0/tmp location where the connect
        worker will store offsets
6. Create a worker.properties and oracle.properties file in confluentinc-kafka-connect-jdbc-10.2.0/ location
7. ```bash
cd /{Path to the kafka connect extracted folder}/confluentinc-kafka-connect-jdbc-10.2.0/
mkdir tmp
touch tmp/connect.offsets
touch worker.properties
touch oracle.properties
```
    a. worker.properties
    ```
    bootstrap.servers=127.0.0.1:9092
    group.id=connect-cluster-2
    key.converter=org.apache.kafka.connect.storage.StringConverter
    value.converter=org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable=true
    value.converter.schemas.enable=true
    internal.key.converter=org.apache.kafka.connect.storage.StringConverter
    internal.value.converter=org.apache.kafka.connect.json.JsonConverter
    internal.key.converter.schemas.enable=false
    internal.value.converter.schemas.enable=false
    offset.storage.topic=connect-offsets-2
    offset.storage.replication.factor=1
    config.storage.topic=connect-configs-2
    config.storage.replication.factor=1
    status.storage.topic=connect-status-2
    status.storage.replication.factor=1
    offset.flush.interval.ms=10000
    rest.host.name=127.0.0.1
    rest.port=8084
    
    offset.storage.file.filename=/{Path to the kafka connect extracted folder}/confluentinc-
    kafka-connect-jdbc-10.2.0/tmp/connect.offsets
    
    plugin.path=/{Path to the kafka connect extracted folder}/confluentinc-kafka-connect-
    jdbc-10.2.0
    ```
    b. oracle.properties
    ```
    name=oracle-connector
    connector.class=io.confluent.connect.jdbc.JdbcSinkConnector
    tasks.max=1
    topics=test
    key.converter=org.apache.kafka.connect.storage.StringConverter
    key.converter.schemas.enable=true
    value.converter=org.apache.kafka.connect.json.JsonConverter
    value.converter.schemas.enable=true
    dialect.name=OracleDatabaseDialect
    connection.url=jdbc:oracle:thin:@//sl73commdb7q-scan:1521/COMM70Q
    connection.user=VBOX750_VRD_SRVR
    connection.password=password
    table.name.format=TVPPR_QTZ_LOCKS
    insert.mode=insert
    pk.mode=none
    auto.create=false
    auto.evolve=false
    ```
    c. In worker.properties make replace the correct paths for offset.storage.file.filename and plugin.path
    ```
    offset.storage.file.filename=/{Path to the kafka connect extracted folder}/confluentinc-
    kafka-connect-jdbc-10.2.0/tmp/connect.offsets
 
    plugin.path=/{Path to the kafka connect extracted folder}/confluentinc-kafka-connect-
    jdbc-10.2.0
    ```
    **Note: DO NOT GIVE THE PLUGIN PATH AS /path to extracted folder/confluentinc-kafka-connect-
    jdbc-10.2.0/lib**, this will result in connector throwing this error "Caused by:
    org.apache.kafka.connect.errors.ConnectException: java.sql.SQLException: No suitable driver found
    for jdbc:oracle:"
8. Make sure your directory roughly looks like this
```
     /{Path to the kafka connect extracted folder}/confluentinc-kafka-connect-jdbc-10.2.0/
        lib/
            kafka-connect-jdbc-10.2.0.jar ojdbc8.jar
        tmp/
        connect.offsets
        worker.properties
        oracle.properties
```
9. To start the connector worker in standalone mode use  
```
cd /{Path to the kafka connect extracted folder}/confluentinc-kafka-connect-jdbc-10.2.0/
connect-standalone worker.properties oracle.properties
```

If you encounter errors such as "Caused by: org.apache.kafka.connect.errors.ConnectException:
java.sql.SQLException: No suitable driver found for jdbc:oracle:", it might be due to the following reasons
* plugin.path in worker.properties is wrong (it should be /{path}/confluentinc-kafka-connect-jdbc-10.2.0 instead of /{path}/confluentinc-kafka-connect-jdbc-10.2.0/lib)
* ojdbc8.jar is missing from /{path}/confluentinc-kafka-connect-jdbc-10.2.0/lib
