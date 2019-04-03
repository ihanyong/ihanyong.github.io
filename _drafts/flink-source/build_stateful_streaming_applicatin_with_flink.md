build_stateful_streaming_applicatin_with_flink


https://www.infoworld.com/article/3293426/big-data/how-to-build-stateful-streaming-applications-with-apache-flink.html

How to build stateful streaming application with apache flink



Take adavantage of Flink's of DataStream API, ProcessFunctions , and SQL support to build event-driven or streaming analytics applications


Apache Flink is a framework for implementing stateful stream processing application and running then at scale on a compute cluster. In a previous article we examined what stateful steram processing is, what use case it addresses, and why you should implement and run your streaming applications with Apache Flink.

In this article, I will present examples for tow common use cases of stateful stream processing and discuss how they can be implemented with Flink. The first use case is event-driven applications , i.e., application that ingest continuous streams of events and apply some business logic to these events. The second is the streaing analytics use case, where I will present tow analytical queries implemented with Flink's SQL API, which agregate streaming data in real-time. We at Data Artisans provide the source code of all of examples in a public GitHub repository.


Before we dive into the details of the examples, I will intorduce the event stream that is ingested by the example application and explain how you can run the code that we provide.

# A stream of taxi ride events

Our example applications are based on a public data set about taxi rides that happened in New York City in 2013. The organizers of the 2015 DEBS Grand challenge rearranged the original data set and converted it into single CSV file from which we care reading the follwing nice fields. 

- Medallion - an MD5 sum id of the taxi
- Hack_license - an MD5 sum id of the taxi license
- Pickup_datetime
- Dropoff_datetime
- Pickup_longitude
- picup_latitude
- Dropoff_longitude
- Dropoff_latitude
- Total_Amount

The CSV file stores the records in ascending order of their drop-off time attribute. hence, the file can be treated as an ordered log of events that were published when a trip ended. In order to run the examples that we provide on GitHub, you need to download the data set of the DEBS challenge from Google Drive.

All example applications sequentially read the CSV file and ingest it as a stream of taix ride events. From there on, the application process the events just like any other stream, i.e., like a stream that is ingested from a log-based publish-subsribe system, such a Apache Kafka or Kinesis. In fact, reading a file(or any other type of perissted data) and treating it as a stream is cornerstone of Flink's approach to unifying batch and stream processing.

# Running the Flink examples

As mentioned earlier, we published the source code of our example application in a GitHub repository, We encourage you to fork and clone the repository. The examples ca be easily executed from within your IDE of choice; you don't need to set up and configure a Flink cluster to run them. First, improt the source code of the examples as a Maven project. Then, execute the main class of an application and provide the storage location of the data file as a program paramenter.

Once you have launched an application, it will start a local, embedded Flink instance inside the application's JVM process and submint the application to  execute it. You will see a bouch of log statements while Flink is starting and the job's tasks are being scheduled. Once the application is running, its output will be writeen to the standard output.


# Building an event-driven application in Flink
Now, let discuss our first use case, which is an event-driven application. Event-driven applications ingest streams of events, perform computations as the evnts are received, and may emit new events or trigger external actions. Multiple event-driven applications can be composed by connecting the together via event log sysytems, similar to how large systems can be composed from microservices. Event-driven applications, event logs, and application state snapshots comprise a very powerful design pattern because you ca reset their state and replay their input to recover from a failure, to fix a bug, ro to migrate an application to a different cluster.

In this article we will examine an event-driven application that backs a service, which monitors the working hours of taxi drivers. In 2016, the NYC taxi and limousine commission decided to restrict the working hours of taxi drivers to 12 hour shifts and require a break of at least eight hours before the next shift may be started. A shift starts with the beginning of the first ride. From then on, a driver may start new rides within a window of 12 hours. Our application tracks the rides of drivers, marks the endtime fo their 12-hour window(i.e., the time when they may start the last ride), and flags rides that violated the regulation. You can find the full source code of this exmaple in our GitHub repository.


Our application is implemented with Flink's DataStream API and a KeyedProcessFunction. the dataStream API is a functional API and based on the concept of typed data streams. a DataStream<T> is the logical representation of a stream of events of type T. A stream is processed by applying a fuction to it that produces another data stream, possibly of a different type. Flink processes streams in parallel by distributing events to stream partitions and applying different instances of functions to each partition.

The following code snippet shows the high-level flow of our monitoring applications.

```java
DataStream<TzxiRide> rides = TaxiRides.getRides(env, inputPath);

DataStream<Tuple2<String, String>> notifications = rides
    .keyBy(r -> r.licenseId)
    .process(new MonitorWorkTime());

notifications.print();

```

The application starts ingesting a stream of taxi ride events. In our example, the events are read from a text file, parsed, and stored in TzxiRide POJO objects. A real-world application would typically ingest the evnts from a message queue or event log, such a Apache Kafka or Pravega. The next step is to key the TaxiRide events by the licenseId of the Driver. The KeyBy operation pratitions the stream on the declared field, suc that all events with the same key are processed by the same parallel instance of the following function. In our case, we partition on the licenseId field becase we want to monitor the working time of each individual driver.

Next, we apply the MonitorWorkTime function on the partitioned TaxiRide events. The function tracks the rides per driver and monitors their shifts and break times. It emits events of type Tuple2<String, String>, where each tuple represents a notification consisting of the license Id of the driver and a message. Finally, our application emits the messages by printing them to the standard output. A real-world application would write the notifications to an external message or storage system, like Apache Kafka, HDFS, or a database system, or would trigger an external call to immediately push them out.

Now that we've discussed the overall flow of the application, let's have a look at the MonitorWorkTime function, which cntains most of the application's actual business logic. The MonitorWorkTime function is a stateful KeyedPorcessFunction that ingests TaxiRide events and emits Tuple2<String, String> records. The KeyedProcessFunction interface features tow methods to process data: processElement() and onTimer(). The processElement() method is called for each arriveing event. The processElement() method is called for each arriving event. The onTime() method is called when a previously registered time fires. The following snippet shows the skeleton of the MonitorWorkTime function and everything that is declared outside of the processing methods.

```java
public static class MonitorWorkTime
    extends KeyedProcessFunction<String, TaxiRide, Tuple2<String, String>> {

    private static final long allowed_work_time = 12 * 60 * 60 * 1000;
    private static final long req_break_time = 8 * 60 * 60 * 1000;
    private static final long clean_up_interval = 28 * 60 * 60 * 1000；

    private transient DateTimeFormatter formatter;

    ValueState<Long> shiftStart;

    @Override
    public void open(Configuration conf) {

        shiftStart = getRuntimeContext().getState(
            new ValueStateDescriptor<>("shiftStart", Types.Long));
        this.formatter = DataTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss");
    }
}
```
The function decalres a few constants for time intervals in milliseconds, a time formatter, and a state handle for keyed state that is managed by Flink. Managed stae is periodically checkpointed and automatically restored in case of a failure. Keyed state is origanized per key, which means that a function will maintain one value per handle and key. In our case, the MonitorWorkTime function maintains a Long value for each key, i.e., for each licenseId. The state handle is initialized in the open() method, which is called once before the first event is processed.


Now, let's have a look at the processElement() method.

```java
public static class MonitorWorkTime extends KeyedProcessFunction<String, TaxiRide, Tuple2<String, String>> {

    @Override
    public void processElement(
        TaxiRide ride, 
        Context ctx, 
        Collector<Tuple2<String, String>> out ) throws Exception {
        Long startTs = shiftStart.value();
        if(startTs == null ||
            startTs < ride.pickUpTime - (allowed_work_time + req_break_time)) {

            startTs = ride.pickUpTime;
            shiftStart.update(startTs);
            long endTs = startTs + allowed_work_time;
            out.collect(Tuple2.of(ride.licenseId,"you are allowed to accept new passengers until ***datatime***"));

            ctx.timerService().registerEventTimeTimer(startTs + clean_up_interval);

        } else if(startTs < ride.pickUpTime - allowed_work_time) {
            out.collect(Tuple2.of(ride.licenseId, "this ride violated the working time regulations."));
        }

    }

    @Override
    public void onTimer() throw Exception {
        Long startTs = shiftStart.value();
        if(startTs == timeerTs - clean_up_interval) {
            shiftStart.clear();
        }
    }
}

```

Teh processElement() method is called for each TaxiRide event. First, The method fetches the start time of the drive'rs shift from the state handle. If the state does not contain a start time or if the last shift started more than 20 hours earlier than the current ride, the current ride is first ride of a new shift. In either case the function start a new shift by updating the start time of the shift to the start time of the current ride, emits a message to the driver with the end time of the new shift, and registers a timer to clean up the state in 24 hours.

If the current ride is not the first ride of a new shift, the function checks if it violates the working time regulation, i.e., whether it started more than 12 hours later than the start of the driver's current shift. If that is the case, the function emits a message to inform the driver about the violation.

The processElement() method of the MonitorWorkTime function registers a timer to clean up the state 24 hours after the start of a shift. Removing state that is no longer needed is important to prevent growing state sizes due to leaking state. A timer fires when the time of the application passes the timer's timestamp. Ath that points, the onTimer() method is called. Similar to state, timers are maintained per key, and the function is put into the context of the associated key before the onTimer() method is called. Hence, all state access is directed to the key that was active when the timer was registered. 

Cleaning up the state is the only logic that the onTimer() method implements. When a timer fires, we check if the driver started a new shift in the meantime, i.3., whether the shift starting time changeed. if that is not the case, we clear the shift state for the driver.

The implementaiton of the working hour monitoring example demonstartes how Flink applications operate with state and time, the core ingredients of any slightly advanced stream processing application. Flink provides many features for working with of state and time, such as support for processing and event time. In addition, Flink provides several strategies for dealing with late records, different types of state, an efficient checkpointing mechanism to guarantee exactly-once state consistency, and many unique features centered around the concept of "savepoints," just to name a rew. The combination of all of these features results in a stream processor that provides the flexibility that is required to build very sophisticated event-driven applications and run then at scale and with operational ease.


# Analyzing data streams with SQL in Flink 
[todo]














