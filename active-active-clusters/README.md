# Active-Active clusters

The idea of this demo is to create two clusters where data is produced in both clusters and consumers consume data generated in both clusters. Cluster and schema linking will be used to send data from left cluster to right cluster and another link will send data from right to left.

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
    curl -s -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8085/subjects/product-value/versions
    curl -s -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8086/subjects/product-value/versions
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
    curl -s http://localhost:8085/subjects/product-value/versions/1 | jq
    curl -s http://localhost:8086/subjects/left.product-value/versions/1 | jq
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
    curl -s http://localhost:8086/subjects/product-value/versions/1 | jq
    curl -s http://localhost:8085/subjects/right.product-value/versions/1 | jq
```

## Create the cluster linking

### Create cluster linking from left to right

1. Create config file to configure the cluster linking
```shell
docker-compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
cluster.link.prefix=left.
consumer.offset.sync.enable=false 
" > /home/appuser/cl.properties'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker-compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link left-to-right-cl \
    --config-file /home/appuser/cl.properties
``` 

3. Create the mirroring
```shell
    docker-compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic left.product \
    --link left-to-right-cl \
    --bootstrap-server rightKafka:29092        
``` 

4. Verifying cluster linking is up
```shell
    docker-compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link left-to-right-cl --list
 ````

Output is similar to `Link name: 'left-to-right-cl', link ID: 'CdDrHuV5Q5Sqyq0TCXnLsw', remote cluster ID: 'nBu7YnBiRsmDR_WilKe6Og', local cluster ID: '1wnpnQRORZ-C2tdxEStVtA', remote cluster available: 'true'`

### Create cluster linking from right to left

1. Create config file to configure the cluster linking
```shell
docker-compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
cluster.link.prefix=right.
consumer.offset.sync.enable=false 
" > /home/appuser/cl.properties'
```

2. Create the cluster link on the *destination* cluster. We are using some extra [configuration options](https://docs.confluent.io/platform/current/multi-dc-deployments/cluster-linking/configs.html#configuration-options).
```shell
    docker-compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link right-to-left-cl \
    --config-file /home/appuser/cl.properties
``` 

3. Create the mirroring
```shell
    docker-compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic right.product \
    --link right-to-left-cl \
    --bootstrap-server leftKafka:19092        
``` 

4. Verifying cluster linking is up
```shell
    docker-compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 --link right-to-left-cl --list
 ````

Output is similar to `Link name: 'right-to-left-cl', link ID: 'CdDrHuV5Q5Sqyq0TCXnLsw', remote cluster ID: 'nBu7YnBiRsmDR_WilKe6Og', local cluster ID: '1wnpnQRORZ-C2tdxEStVtA', remote cluster available: 'true'`
