# Bi-directional cluster link

## Start the clusters

```shell
    docker compose up -d
``` 

Two CP clusters are running:

*  Left Control Center available at [http://localhost:19021](http://localhost:19021/)
*  Right Control Center available at [http://localhost:29021](http://localhost:29021/)
*  Left Schema Register available at [http://localhost:8085](http://localhost:8085/)
*  Right Schema Register available at [http://localhost:8086](http://localhost:8086/)

## Create the topic `product` and the schema `product-value` in the both clusters

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

## Checking the link is the same

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

### Produce in the main cluster and consume in the main and disaster

1. Producer produces to left cluster
```shell
   docker compose exec leftKafka kafka-console-producer \
    --bootstrap-server leftKafka:19092 \
    --topic product
    >riceLeft
    >beansLeft
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

3. Consumer consumes from right cluster

```shell
    docker compose exec rightKafka kafka-console-consumer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --group right-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

5. Checking the consumer offsets

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group right-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
```
Offsets are synchronised.

### Reverse to disaster cluster

Check the status

```shell
    docker compose exec rightKafka \
    kafka-mirrors --bootstrap-server rightKafka:29092 \
    --describe --topics product

    docker compose exec leftKafka \
    kafka-mirrors --bootstrap-server leftKafka:19092 \
    --describe --topics product
```


Launch the command to start the reverse

```shell
    docker compose exec rightKafka \
    kafka-mirrors --reverse-and-start \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092
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

Producer produces to right cluster
```shell
   docker compose exec rightKafka kafka-console-producer \
    --bootstrap-server rightKafka:29092 \
    --topic product
    >riceRight
    >beansRight
```

Consumer consumes from right cluster

```shell
    docker compose exec rightKafka kafka-console-consumer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --group right-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

Checking the offsets

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group right-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group right-group
```

### Reverse to original configuration (main -> disaster)

Check the status

```shell
    docker compose exec rightKafka \
    kafka-mirrors --bootstrap-server rightKafka:29092 \
    --describe --topics product

    docker compose exec leftKafka \
    kafka-mirrors --bootstrap-server leftKafka:19092 \
    --describe --topics product
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
    >riceLeft2
    >beansLeft2
```

Consumer consumes from right cluster

```shell
    docker compose exec rightKafka kafka-console-consumer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --group right-group \
    --from-beginning \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.value=true
```

Checking the offsets

```shell
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group left-group
    docker compose exec leftKafka kafka-consumer-groups --bootstrap-server leftKafka:19092 --describe --group right-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group left-group
    docker compose exec rightKafka kafka-consumer-groups --bootstrap-server rightKafka:29092 --describe --group right-group
```
