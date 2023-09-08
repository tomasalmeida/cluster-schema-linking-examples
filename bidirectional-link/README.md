# Bi-directional cluster link

## Start the clusters

```shell
    docker-compose up -d
``` 

Two CP clusters are running:

*  Left Control Center available at [http://localhost:19021](http://localhost:19021/)
*  Right Control Center available at [http://localhost:29021](http://localhost:29021/)
*  Left Schema Register available at [http://localhost:8085](http://localhost:8085/)
*  Right Schema Register available at [http://localhost:8086](http://localhost:8086/)

## Create the topic `product` and the schema `product-value` in the both clusters

###  Create the schema `product-value` 

```shell
    curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8085/subjects/product-value/versions
    curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8086/subjects/product-value/versions     
```

###  Create the topic `product` 
```shell
    docker-compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic product --create --partitions 1 --replication-factor 1
    docker-compose exec rightKafka kafka-topics --bootstrap-server rightKafka:29092 --topic product --create --partitions 1 --replication-factor 1
```

## Create the schema linking

### Create schema linking from left to right

1. Create config file 
```shell
    docker-compose exec leftSchemaregistry bash -c '\
    echo "schema.registry.url=http://rightSchemaregistry:8086" > /home/appuser/config.txt'
```

2. Create the schema exporter 
```shell
    docker-compose exec leftSchemaregistry bash -c '\
    schema-exporter --create --name left-to-right-sl --subjects "product-value" \
    --config-file ~/config.txt \
    --schema.registry.url http://leftSchemaregistry:8085 \
    --subject-format "left.product-value" \
    --context-type NONE'
```

3. Validate exporter is working
```shell
    docker-compose exec leftSchemaregistry bash -c '\
    schema-exporter --list \
    --schema.registry.url http://leftSchemaregistry:8085'
````

4. Check the exporter is running
```shell
    docker-compose exec leftSchemaregistry bash -c '\
    schema-exporter --get-status --name left-to-right-sl --schema.registry.url http://leftSchemaregistry:8085' | jq
```

5. Check the schema is the same in the left and right cluster (right cluster has the schema with a different name)

```shell
    curl http://localhost:8085/subjects/product-value/versions/1 | jq
    curl http://localhost:8086/subjects/left.product-value/versions/1 | jq
```

### Create schema linking from right to left

1. Create config file 
```shell
    docker-compose exec rightSchemaregistry bash -c '\
    echo "schema.registry.url=http://leftSchemaregistry:8085" > /home/appuser/config.txt'
```

2. Create the schema exporter 
```shell
    docker-compose exec rightSchemaregistry bash -c '\
    schema-exporter --create --name right-to-left-sl --subjects "product-value" \
    --config-file ~/config.txt \
    --schema.registry.url http://rightSchemaregistry:8086 \
    --subject-format "right.product-value" \
    --context-type NONE'
```

3. Validate exporter is working
```shell
    docker-compose exec rightSchemaregistry bash -c '\
    schema-exporter --list \
    --schema.registry.url http://rightSchemaregistry:8086'
````

4. Check the exporter is running
```shell
    docker-compose exec rightSchemaregistry bash -c '\
    schema-exporter --get-status --name right-to-left-sl --schema.registry.url http://rightSchemaregistry:8086' | jq
```

5. Check the schema is the same in the left and right cluster (right cluster has the schema with a different name)

```shell
    curl http://localhost:8086/subjects/product-value/versions/1 | jq
    curl http://localhost:8085/subjects/right.product-value/versions/1 | jq
```

## Create the cluster linking

### Create cluster linking from left to right

1. Create config file to configure the cluster linking
```shell
docker-compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
" > /home/appuser/cl.properties'

docker-compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker-compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
``` 

3. Create the mirroring
```shell
    docker-compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic left.product \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092        
``` 

4. Verifying cluster linking is up
```shell
    docker-compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list
 ````

Output is similar to `Link name: 'bidirectional-link', link ID: 'AMw2VoNJRya3CgMHGRnwIQ', remote cluster ID: 'JZOncuJyRQqaQ_qt8Mi_UA', local cluster ID: '_-6HdK0dS_S6Z9xZO8usOg', remote cluster available: 'true', link state: 'ACTIVE'`

### Create cluster linking from right to left

1. Create config file to configure the cluster linking
```shell
docker-compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
cluster.link.prefix=right.
consumer.offset.sync.enable=true 
" > /home/appuser/cl2.properties'

docker-compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker-compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json
``` 

3. Create the mirroring
```shell
    docker-compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic right.product \
    --link bidirectional-link \
    --bootstrap-server leftKafka:19092
``` 

4. Verifying cluster linking is up
```shell
    docker-compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 --link bidirectional-link --list
 ````

Output is similar to `Link name: 'bidirectional-link', link ID: 'AMw2VoNJRya3CgMHGRnwIQ', remote cluster ID: '_-6HdK0dS_S6Z9xZO8usOg', local cluster ID: 'JZOncuJyRQqaQ_qt8Mi_UA', remote cluster available: 'true', link state: 'ACTIVE'`

## Checking the link is the same

### Check again the created links

```shell
 docker-compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --link bidirectional-link --list
docker-compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list
```

Verifying the results:
- **Link name:** 'bidirectional-link' -> same name for both results, when the links were created, the same name was given to them
- **link ID:** 'AMw2VoNJRya3CgMHGRnwIQ' -> same id in both links
- **remote cluster ID:** '_-6HdK0dS_S6Z9xZO8usOg' and 'JZOncuJyRQqaQ_qt8Mi_UA' are shown in a crossed way
- **local cluster ID:** 'JZOncuJyRQqaQ_qt8Mi_UA' and '_-6HdK0dS_S6Z9xZO8usOg' are shown in a crossed way

## Test time!

### Active-active

1. Producer produces to left cluster
```shell
   docker-compose exec leftSchemaregistry kafka-avro-console-producer \
    --bootstrap-server leftKafka:19092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://leftSchemaregistry:8085 \
    --property auto.register=false \
    --property use.latest.version=true

    { "product_id": 1, "product_name" : "riceLeft"} 
    { "product_id": 2, "product_name" : "beansLeft"} 
```


2. Consumer consumes from left cluster

```shell
    docker-compose exec leftSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server leftKafka:19092 \
        --property schema.registry.url=http://leftSchemaregistry:8085 \
        --group disaster-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```


3. Producer produces to right cluster
```shell
   docker-compose exec rightSchemaregistry kafka-avro-console-producer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://rightSchemaregistry:8086 \
    --property auto.register=false \
    --property use.latest.version=true

    { "product_id": 1, "product_name" : "riceRight"} 
    { "product_id": 2, "product_name" : "beansRight"} 
```


4. Consumer consumes from right cluster

```shell
    docker-compose exec rightSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server rightKafka:29092 \
        --property schema.registry.url=http://rightSchemaregistry:8086 \
        --group left-groupRight \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```

### disaster mode


### Dummy mode (same consumer group in both sides)