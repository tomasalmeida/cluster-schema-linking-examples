# cluster-schema-linking-examples

Examples of cluster and schema linking.

## [source-to-destination](source-to-destination/)

The idea of this demo is to create a source cluster and a destination cluster, the `product` schema and data are created in the source cluster/schema registry and it is replicated to the destination cluster using schema and cluster linking. We make use of schema contexts to isolate subjects and topic prefixing.

## [disaster-recovery](disaster-recovery/)

The idea of this demo is to create a main cluster and a disaster recovery cluster, the `product` schema and data are created in the main cluster/schema registry and it is replicated to the disaster recovery cluster using schema and cluster linking. We then, stop the main cluster, move consumers and producers to disaster recovery, restart the main cluster and move the data back
