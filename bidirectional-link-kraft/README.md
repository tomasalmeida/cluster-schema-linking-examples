# Bi-directional Cluster Link 

## Start the clusters

```shell
    docker compose up -d
``` 

Two CP clusters are running:

*  Left Control Center available at [http://localhost:19021](http://localhost:19021/)
*  Right Control Center available at [http://localhost:29021](http://localhost:29021/)

## Create the topic `product` in the left cluster

###  Create the topic `product` 
```shell
    docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic product --create --partitions 1 --replication-factor 1
```

## Create the cluster linking

### Create cluster linking from left to right

1. Create config file to configure the cluster linking
```shell
docker compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
consumer.offset.sync.enable=true
" > /home/appuser/cl.properties'

docker compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
``` 

3. Create the mirroring
```shell
    docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic product \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092        
``` 

4. Verifying cluster linking is up
```shell
    docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list
 ````

Output is similar to `Link name: 'bidirectional-link', link ID: 'AMw2VoNJRya3CgMHGRnwIQ', remote cluster ID: 'JZOncuJyRQqaQ_qt8Mi_UA', local cluster ID: '_-6HdK0dS_S6Z9xZO8usOg', remote cluster available: 'true', link state: 'ACTIVE'`


### Create cluster linking from right to left

1. Create config file to configure the cluster linking
```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
consumer.offset.sync.enable=true
" > /home/appuser/cl2.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json
``` 

3. Verifying cluster linking is up
```shell
    docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 --link bidirectional-link --list
 ````

Output is similar to `Link name: 'bidirectional-link', link ID: 'AMw2VoNJRya3CgMHGRnwIQ', remote cluster ID: '_-6HdK0dS_S6Z9xZO8usOg', local cluster ID: 'JZOncuJyRQqaQ_qt8Mi_UA', remote cluster available: 'true', link state: 'ACTIVE'`

## Checking the link is the same in both sides

### Check again the created links

```shell
    docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --link bidirectional-link --list
    docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list
```

Verifying the results:
- **Link name:** 'bidirectional-link' -> same name for both results, when the links were created, the same name was given to them
- **link ID:** 'AMw2VoNJRya3CgMHGRnwIQ' -> same id in both links
- **remote cluster ID:** '_-6HdK0dS_S6Z9xZO8usOg' and 'JZOncuJyRQqaQ_qt8Mi_UA' are shown in a crossed way
- **local cluster ID:** 'JZOncuJyRQqaQ_qt8Mi_UA' and '_-6HdK0dS_S6Z9xZO8usOg' are shown in a crossed way

## Test time!

## Active/Passive

### Produce and consume in the left cluster

1. Producer produces to left cluster
```shell
   docker compose exec leftKafka kafka-console-producer \
    --bootstrap-server leftKafka:19092 \
    --topic product
```

2. Consumer consumes from left cluster

```shell
    docker compose exec leftKafka kafka-console-consumer \
    --bootstrap-server leftKafka:19092 \
    --topic product \
    --group left-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

3. Checking the consumer offsets are synchronised in both clusters

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
```
Offsets are synchronised.

## Simulating a disaster

### Stop left cluster (and all consumer or producers you have created)
```
    docker-compose stop leftKafka  leftSchemaregistry leftControlCenter
```

## FAILOVER: Promote disaster cluster to principal cluster (right cluster)

### Check the status of the mirror topic
```
    docker-compose exec rightKafka \
        kafka-mirrors --bootstrap-server rightKafka:29092 \
        --describe --topics product 
```

### Promote topic to writable

1. Stop mirroring

Note: we are using `--failover` because the main cluster is unavailable, if we want to sync before promoting the topic, we should use the option `--promote` instead.


```
    docker-compose exec rightKafka \
        kafka-mirrors --bootstrap-server rightKafka:29092 \
        --failover --topics product
```

2. Verify that the mirror topic is not a mirror anymore
```
    docker-compose exec rightKafka \
        kafka-mirrors --bootstrap-server rightKafka:29092 \
        --describe --topics product 
```

The result should have the `State: STOPPED` as part of it.

###  Produce some data now in the right cluster

```
   docker compose exec rightKafka kafka-console-producer \
    --bootstrap-server rightKafka:29092 \
    --topic product
```

### Consume data from the right cluster (DR) with the same consumer group (left-group)
```
docker compose exec rightKafka kafka-console-consumer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --group left-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

### FAILOVER complete

## FAILBACK Promote left cluster as main cluster again (failback)

### Start left cluster

```
    docker-compose start leftKafka  leftSchemaregistry leftControlCenter
```

### Reverse to original configuration (left -> right (DR) )

Check the status of the topics in both clusters

```shell
    docker compose exec rightKafka \
    kafka-mirrors --bootstrap-server rightKafka:29092 \
    --describe --topics product

    docker compose exec leftKafka \
    kafka-mirrors --bootstrap-server leftKafka:19092 \
    --describe --topics product
```

Launch the command to restore the topic from the left cluster from the promoted right (DR) topic.
```
    docker compose exec leftKafka kafka-mirrors --bootstrap-server leftKafka:19092  --truncate-and-restore --topics product --link bidirectional-link
```

Checking the consumer offsets are synchronised in both clusters

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
```

Launch the command to start the reverse

```shell
    docker compose exec leftKafka \
    kafka-mirrors --reverse-and-start \
    --link bidirectional-link \
    --bootstrap-server leftKafka:19092
```

Check the results

```shell
    docker compose exec rightKafka \
    kafka-mirrors --bootstrap-server rightKafka:29092 \
    --describe --topics product

    docker compose exec leftKafka \
    kafka-mirrors --bootstrap-server leftKafka:19092 \
    --describe --topics product
```

Producer produces to left cluster

```shell
  docker compose exec leftKafka kafka-console-producer \
    --bootstrap-server leftKafka:19092 \
    --topic product
```

Consumer consumes from left cluster

```shell
    docker compose exec leftKafka kafka-console-consumer \
    --bootstrap-server leftKafka:19092 \
    --topic product \
    --group left-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

Checking the offsets

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
```


