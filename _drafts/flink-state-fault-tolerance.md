Stateful function and operators store data across the processing of individual elements/events, making state a critical building block for any type of more elaborate opreation.

- When an application searches for cetain evetn patterns, the state will store the sequence of events encountered so far.
- When aggregating events per minute/hour/day, the state holds the pending aggregates.
- When training a machine learing model over a steram of data points, the state holds the current version of the model parameters.
- When historic data needs to be managed, the state allows efficient access to events that occureed in the past



- rescaling   
- queryable state

main content
- working with state
- the broadcast state pattern
- checkpointing
- queryable state
- state backends
- state schema evolution
- custom state serialization


# working with state
- Keyed state and Operator state
- Raw and managed state
- using managed keyed state
- using managed Operator state

Keyed State is always relative to keys and can only be used in functions and operators on a keyedStream.

With Operator State, each operator state is bound to one parallel operator instance.

## Using Managed Keyed State
- ValueState<T>
- ListState<T>
- ReducingState<T>
- AggregatingState<IN,OUT>
- FoldingState<T, ACC>
- MapState<UK,UV>

keep in mind
- state objects are only used for interfacing with state.
- the value you get from the state depends on the key of the input element.

State is accessed using the RuntimeContext, which is only possible in rich functions.


state Time-To-Live TTL
exipired state won't be removed until is read, possibly leading to ever growing state.

timeService to clean a state? 


## Using Managed Operator State
```java
public interface CheckpointedFunction {
    // whenever a checkpoint has to be performed, snapshotState() is called.
    void snapshotState(FunctionSnapshotContext context) throws Exception;
    // is called every time the user-defined function is initialized.
    // initialized and recovery logic are here
    void initializeState(FunctionInitializationContext context) throws Exception
}
```

redistribution schemes
- Event-split redistribution:  each operator gets a sublist.
- Union redistribution: each opeartor get the complete list of state elements

```java
public interface OperatorStateStore {
    // 
    <S> ListState<S> getListState(ListStateDescriptor<S> stateDescriptor) throws Exception;
    // 
    <S> ListState<S> getUnionListState(ListStateDescriptor<S> stateDescriptor) throws Exception;

}
```

FunctionInitializationContext.isRestored();  

#### listCheckPoined interface
only supports list style state with even-split redistribution scheme on restore.

#### CheckpointListener

### Stateful Source Funcitons


# The Broadcast State Pattern

some data coming from one stream is required to be broadcasted to all donstream tasks, where it is stored locally and is used to process all incomming elements on the other stream.

1. it has a map format
2. it is only available to specific operators that have as inputs a broadcasted stream and a non-broadcasted one , and 
3. such a operator can have multiple broadcast states with different names.


### using
- broadcaststream(broadcast, connected, process)

important condiderations
- there i no cross-task communication
- Order of events in broadcast state may differ across tasks
- all tasks checkpoint their broadcast state
- No rocksDB state backend

# queryable sate beta

### Architecture
1. the QueryableStateClient
2. QueryableStateeClientProxy
3. QueryableStateServer

### activating queryable state
copy the flink-queryable-state-runtime_XXX.jar from the opt/ folder to lib/ folder

### making state queryable
- QueryableStream
- Managed Keyed State-> descriptor.setQueryable("query-name");
### Limitations
- The queryable state life-cycle is bound to the life-cycle of the job,  e.g. tasks register queryable state on startup and unregister it on disposal. In future versions, it is desirable to decouple this in order to allow queries after a task finishes, and to speed up recovery via state replication.
- Notifications about available KvState happpen via a simple tell. In the future this should be imporved to be more robust with asks and acknowledgements.
- the server and client keep track of statistics for queries. These are currently disabled by default as the would not be exposed anywhere. As soon as there is better wupport to publish these numbers via the Metrics system, we should enable the stats.

# Evolving state schema

steps:
1. take a savepoint of flink streaming job.
2. update state byeps in application (e.g., modifying Avro Type Schema)
3. restore the job from the savepoint. When accessing state for the first time, Flink will assess whether or not the schema had been changed for the state, and migrate stateschema if necessary.




