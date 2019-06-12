elasticsearch_study.md


# Document APIs
## Reading and Writing documents

- index divided into shards
- shard can have multiple copies -> replication group
- keep replication group in sync -> data replication model


data replicatoin model based on the primary-backup model 

### Basic write model

- every indexing opreation is first resolved to a replication group (shards) using routing based on the document ID.  
- Then the operation is forwarded to the primary shard of the group(routing on first step). 
- Primary shard is responsible for validating the operation and forwading it to the other replicas. 
- replicas ca be offline -> in-sync copies maintained by the master node.

basic flow
    1. Validate incoming operation and reject it if structurally invalid
    2. Execute the operation locally
    3. Forward the operation to each replica in the current in-sync copies set(in parallel)
    4. Once all replcas have successfully performed the operation and responded to the primary, the primary acknowledges the successful completion of the request to the client.


#### Failure handling

- Primary fails
- replica shards fails
- primary isolated due to a network partition(or a long GC)


### Basic read model
- lightweight lookups by ID
- heavy search request

basic flow: 
1. Resolve the read requests to the relevant shards
2. select an active copy of each relevants shard
3. send shrad level read requests to eh selected copies
4. combine the result and response


#### Failure handing
- resend the shard level search request to the copy instead. 
- partial results



### A few simple implications
- Efficient reads 
- read unacknowledged
- Two copies by default


### Failures
- A single shard can slow down indexing
- Dirty reads



## Index API

### automatice index creation
dynamic type mapping
```
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "twitter,index10,-index1*,+ind*" 
    }
}

PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "false" 
    }
}

PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "true" 
    }
}
```

### Versioning
     optimistic concurrency control

version_type
    - internal
    - external
    - external_gt
    - external_gte


### Automatic Id Generation
- Post used instead of PUT


### Routing
- /index_name/type_name/?routing=routing_str
- explicit mapping : _routing.   (very minimal) cost of an additional document parsing pass


