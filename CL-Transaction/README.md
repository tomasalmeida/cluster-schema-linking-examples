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

### Verifying consumer group offset is migrated

```shell
docker-compose exec disasterKafka-1 kafka-consumer-groups --bootstrap-server disasterKafka-1:29092 --group test-group --describe

Consumer group 'test-group' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test-group      product         0          2               2               0               -               -               -
```

Same results from source cluster.

### Verifying data is migrated

```shell
docker-compose exec disasterKafka-1 \
        kafka-console-consumer --bootstrap-server disasterKafka-1:29092 \
        --from-beginning \
        --consumer-property "isolation.level=read_committed" \
        --topic product
```

messages are migrated as expected

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

### Produce some data (the last lines with the product data)

```shell
docker-compose exec disasterSchemaregistry \
   kafka-avro-console-producer \
    --bootstrap-server disasterKafka-1:29092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://disasterSchemaregistry:8086 \
    --property auto.register=false \
    --property use.latest.version=true


{ "product_id": 3, "product_name" : "tomato"} 
{ "product_id": 4, "product_name" : "lettuce"}
```

### Find hanging transaction

When the topic is promoted is time to find hanghing transacions

```shell
docker-compose exec disasterKafka-1 \
       kafka-transactions --bootstrap-server disasterKafka-1:29092 find-hanging --broker-id 2  --max-transaction-timeout 5
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
kafka-transactions --bootstrap-server disasterKafka-1:29092 abort --producer-id 200 --start-offset 12 --partition 0 --topic product
```

The consumer on destination will start to consume again.


