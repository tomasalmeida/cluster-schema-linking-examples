# Cluster Linking & Transaction

The idea of this demo is to create a main cluster and a disaster recovery cluster, the `product` topic is created in the main cluster and it is replicated in the disaster recovery cluster using cluster linking. In the main cluster, a producer will produce data using transactions. In the recovery cluster, a consumer (`isolation.level=read_committed`) will read events from `product topic. When the producer is in the main of a transaction we will simulate cluster failover. Then stop the main cluster, promote the mirror topic, find the hanging transaction and abort it. Finally, we move producers to disaster recovery.

## Start the clusters

```shell
docker-compose up -d
```

Two CP clusters are running:

* Main Control Center available at [http://localhost:19021](http://localhost:19021/)
* Disaster Recovery Control Center available at [http://localhost:29021](http://localhost:29021/)
* Main Schema Register available at [http://localhost:8085](http://localhost:8085/)
* Disaster Recovery Control Center available at [http://localhost:8086](http://localhost:8086/)

## Create the topic `product`  in the main cluster

### Create the topic `product` (and another topic)

```shell
docker-compose exec mainKafka-1 kafka-topics --bootstrap-server mainKafka-1:19092 --topic product --create --partitions 1 --replication-factor 1
docker-compose exec mainKafka-1 kafka-topics --bootstrap-server mainKafka-1:19092 --topic other-topic --create --partitions 1 --replication-factor 1
```

## Create the cluster linking (main to disaster cluster)

### Create config file to configure the cluster linking

```shell
docker-compose exec disasterKafka-1 bash -c '\
echo "\
bootstrap.servers=mainKafka-1:19092
consumer.offset.sync.enable=true 
consumer.offset.group.filters="{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}"
" > /home/appuser/cl.properties'
```

### Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).

```shell
docker-compose exec disasterKafka-1 \
    kafka-cluster-links --bootstrap-server disasterKafka-1:29092 \
    --create --link main-to-disaster-cl \
    --config-file /home/appuser/cl.properties
```

### Create the mirroring

```shell
docker-compose exec disasterKafka-1 \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic product \
    --link main-to-disaster-cl \
    --bootstrap-server disasterKafka-1:29092
```

### Verifying cluster linking is up

```shell
docker-compose exec disasterKafka-1 kafka-cluster-links --bootstrap-server disasterKafka-1:29092 --link main-to-disaster-cl --list
````

Output is similar to `Link name: 'main-to-disaster-cl', link ID: 'CdDrHuV5Q5Sqyq0TCXnLsw', remote cluster ID: 'nBu7YnBiRsmDR_WilKe6Og', local cluster ID: '1wnpnQRORZ-C2tdxEStVtA', remote cluster available: 'true'`

### Verifying data is migrated

```shell
docker-compose exec disasterKafka-1 \
        kafka-console-consumer --bootstrap-server disasterKafka-1:29092 \
        --from-beginning \
        --consumer-property "isolation.level=read_committed" \
        --topic product
```

messages are migrated as expected

## Produce transaction data.

Produce data. We are using an application provide by confluent in the following [link](https://developer.confluent.io/courses/architecture/transactions-hands-on/).

```shell
docker exec -it java_app  bash \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt1
```

Press enter five tiemes. The output of the command will be similar to

```
*** Begin Transaction ***
*** transactional.id tt1 ***
Sent 0:2

Sent 1:6

Sent 2:7

Sent 3:7

Sent 4:4

*** Commit Transaction ***
```

Produce data again but this time we will not commit the transaction. For example, just press enter two times.

Validate transaction is ongoing

```shell
docker-compose exec disasterKafka-1 \
kafka-transactions --bootstrap-server mainKafka-1:19092 list
```

Stop coordinator broker.

```shell
docker-compose stop <<mainKafka-1>> mainControlCenter
```

## Simulating a disaster

### Stop main cluster (and all consumer or producers you have created)

```shell
docker-compose stop mainKafka-1 mainKafka-2 mainKafka-3 mainZookeeper mainSchemaregistry mainControlCenter
```

## FAILOVER: Promote disaster cluster to principal cluster

### Promote topic to writable

1. Stop mirroring

Note: we are using `--failover` because the main cluster is unavailable, if we want to sync before promoting the topic, we should use the option `--promote` instead.

```shell
docker-compose exec disasterKafka-1 \
        kafka-mirrors --bootstrap-server disasterKafka-1:29092 \
        --failover --topics product
```

2. Verify that the mirror topic is not a mirror anymore

```shell
docker-compose exec disasterKafka-1 \
        kafka-mirrors --bootstrap-server disasterKafka-1:29092 \
        --describe --topics product
```

The result should have the `State: STOPPED` as part of it.

### Find hanging transaction

When the topic is promoted is time to find hanghing transacions

```shell
docker-compose exec disasterKafka-1 \
kafka-transactions --bootstrap-server disasterKafka-1:29092 find-hanging --topic product --max-transaction-timeout 1
```

Repit this command changing the broker-id
The output should be

```shell
Topic  	Partition	ProducerId	ProducerEpoch	CoordinatorEpoch	StartOffset	LastTimestamp	Duration(min)	
product	0        	2000      	2            	0               	12         	1711021818135	5
```

### Abort hanging transaction

It is time to abort hanging transaction

```shell
docker-compose exec disasterKafka-1 \
kafka-transactions --bootstrap-server disasterKafka-1:29092 abort --producer-id 200 --start-offset 12 --partition 0 --topic product
```


### Produce some data

```shell
docker-compose exec java_app \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt1
```

The consumer on destination will start to consume again.

