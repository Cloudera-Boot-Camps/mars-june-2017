# mars-june-2017
Mission to Mars


# <a name="top"></a> Exercise

1. [Run measurements stream using the provided data generator](#s1)

2. [Build a `generator -> Flume -> HBase` pipeline](#s2)
    * Then switch out HBase for HDFS

3. [Build a `generator -> Flume -> Kafka -> Spark Streaming -> HBase` pipeline](#s3)
    * Then switch out HBase for Kudu

4. [First use built-in Spark-Kudu API, then use Envelope](#s4)
    * Then switch out Kudu for Solr



## <a name="s1"></a> Setting up a pseudo data generator source [^](#top)

The generator simply generates an infinite stream of comma seperated records and writes them to a user defined TCP endpoint.

```java
Socket echoSocket = new Socket(hostName, portNumber);
PrintWriter out   = new PrintWriter(echoSocket.getOutputStream(), true);

while (true) {
    Random random = new Random();
    
    String measurementID = UUID.randomUUID().toString();
    
    int detectorID = random.nextInt(8) + 1;
    int galaxyID = random.nextInt(128) + 1;
    int astrophysicistID = random.nextInt(106) + 1;
    
    long measurementTime = System.currentTimeMillis();
    
    double amplitude1 = random.nextDouble();
    double amplitude2 = random.nextDouble();
    double amplitude3 = random.nextDouble();
    
    String delimiter = ",";
    String measurement = 
    		measurementID.    + delimiter + 
    		detectorID.       + delimiter + 
    		galaxyID.         + delimiter + 
    		astrophysicistID. + delimiter + 
    		measurementTime.  + delimiter +
    		amplitude1        + delimiter + 
    		amplitude2        + delimiter + 
    		amplitude3;
    
    out.println(measurement);
    System.out.println(measurement);
}
```

## <a name="s2"></a> Forwarding generated data to HBase using Flume [^](#top)

### Creating a HBase table [^](#top)

HBase can be configured using the provided shell `hbase shell`. To create a table the command from the Hbase shell (not bash/sh) run:
```
[ec2-user@ip-xxx-xx-xx-xxx ~]$ hbase shell
17/06/30 12:48:08 INFO Configuration.deprecation: hadoop.native.lib is deprecated. Instead, use io.native.lib.available
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.2.0-cdh5.11.1, rUnknown, Thu Jun  1 10:19:43 PDT 2017

hbase(main):001:0> create 'some_table', {NAME => 'a_column_family'}
0 row(s) in 1.7200 seconds

=> Hbase::Table - some_table
hbase(main):002:0> 
```

HBase tables can be utilized for either unstructured data (Key-Value) or structured data (columnar).

### Setting up Flume with TCP socket source and HBase sink [^](#top)

#### Flume configuration with regex tokenization of columns
```
# Name the components on this agent 
Agent1.sources = netcat-source  
Agent1.channels = memory-channel
#Agent1.sinks = logger-sink,hbase-sink
Agent1.sinks = hbase-sink

# Describe/configure Source
Agent1.sources.netcat-source.type = netcat
#Agent1.sources.netcat-source.bind = ec2-xx-xxx-xxx-xx.us-west-2.compute.amazonaws.com
Agent1.sources.netcat-source.bind = 0.0.0.0
Agent1.sources.netcat-source.port = 20170

# Describe the sink
#Agent1.sinks.logger-sink.type = logger
Agent1.sinks.hbase-sink.type= hbase

# Use a channel which buffers events in memory
Agent1.channels.memory-channel.type = memory
Agent1.channels.memory-channel.capacity = 100000
Agent1.channels.memory-channel.transactionCapacity = 100

# Bind the source and sink to the channel
Agent1.sources.netcat-source.channels = memory-channel
#Agent1.sinks.logger-sink.channel = memory-channel

Agent1.sinks.hbase-sink.channel = memory-channel
Agent1.sinks.hbase-sink.table = measurements
Agent1.sinks.hbase-sink.columnFamily = structured

Agent1.sinks.hbase-sink.serializer=org.apache.flume.sink.hbase.RegexHbaseEventSerializer
Agent1.sinks.hbase-sink.serializer.regex=(.+?),(\\d+),(\\d+),(\\d+),(\\d+),(\\d+\\.\\d+),(\\d+\\.\\d+),(\\d+\\.\\d+)
Agent1.sinks.hbase-sink.serializer.colNames=measurement_id,detector_id,galaxy_id,person_id,measurement_time,amp_1,amp_2,amp_3
```

#### Substitute existing memory channel in Flume for Kafka instead

```
# Name the components on this agent 
Agent1.sources = netcat-source  
Agent1.channels = kafka-channel
#Agent1.sinks = logger-sink,hbase-sink
Agent1.sinks = hbase-sink

# Describe/configure Source
Agent1.sources.netcat-source.type = netcat
#Agent1.sources.netcat-source.bind = ec2-34-212-116-12.us-west-2.compute.amazonaws.com
Agent1.sources.netcat-source.bind = 0.0.0.0
Agent1.sources.netcat-source.port = 20170

# Describe the sink
#Agent1.sinks.logger-sink.type = logger
Agent1.sinks.hbase-sink.type= hbase

# Use a channel which buffers events in memory
Agent1.channels.kafka-channel.type = org.apache.flume.channel.kafka.KafkaChannel
Agent1.channels.kafka-channel.capacity = 10000
Agent1.channels.kafka-channel.zookeeperConnect = ec2-34-212-116-12.us-west-2.compute.amazonaws.com:2181
Agent1.channels.kafka-channel.parseAsFlumeEvent = false
Agent1.channels.kafka-channel.topic = yeah
Agent1.channels.kafka-channel.consumer.group.id = channel2-grp
Agent1.channels.kafka-channel.auto.offset.reset = earliest
#Agent1.channels.kafka-channel.bootstrap.servers = ec2-34-212-203-105.us-west-2.compute.amazonaws.com:9092
Agent1.channels.kafka-channel.brokerList = ec2-34-212-203-105.us-west-2.compute.amazonaws.com:9092
Agent1.channels.kafka-channel.transactionCapacity = 1000
Agent1.channels.kafka-channel.kafka.consumer.max.partition.fetch.bytes=209715

# Bind the source and sink to the channel
Agent1.sources.netcat-source.channels = kafka-channel
#Agent1.sinks.logger-sink.channel = memory-channel

Agent1.sinks.hbase-sink.channel = kafka-channel
Agent1.sinks.hbase-sink.table = measurements
Agent1.sinks.hbase-sink.columnFamily = structured
```

## <a name="s3"></a> Utilize Spark Streaming to bridge Kafka with Kudu [^](#top)

```java
@SuppressWarnings("serial")
public static void main(String[] args) throws Exception {
    final String brokersArgument        = args[0];
    final String topicsArgument         = args[1];
    final String kuduConnectionArgument = args[2];
    final String kuduTableArgument      = args[3];
    
    SparkConf sparkConf = new SparkConf();
    
    sparkConf.setMaster("local[*]");
    sparkConf.setAppName("KafkaKuduStreamApp");
    
    JavaSparkContext     sc  = new JavaSparkContext(sparkConf);
    JavaStreamingContext ssc = new JavaStreamingContext(sc, new Duration(5000));
    
    Map<String, String> params = Maps.newHashMap();
    params.put("metadata.broker.list", brokersArgument);
    Set<String> topics = Sets.newHashSet(topicsArgument);
    
    // Create direct kafka stream with brokers and topics
    JavaPairDStream<String, String> dstream = KafkaUtils.createDirectStream(
    		ssc, 
    		String.class, 
    		String.class, 
    		StringDecoder.class, 
    		StringDecoder.class, 
    		params, 
    		topics
    	);
    
    dstream.foreachRDD(new Function<JavaPairRDD<String, String>, Void>() {
        @Override
        public Void call(JavaPairRDD<String, String> batch) throws Exception {
            batch.foreachPartition(new VoidFunction<Iterator<Tuple2<String, String>>>() {
                @Override
                public void call(Iterator<Tuple2<String, String>> batchPartitionIterator) throws Exception {
                	
                    KuduClient client   = new KuduClient.KuduClientBuilder(kuduConnectionArgument).build();
                    KuduSession session = client.newSession();
                    KuduTable table     = client.openTable(kuduTableArgument);

                    while (batchPartitionIterator.hasNext()) {
                        String message = batchPartitionIterator.next()._2();
                        String[] values = message.split(",");
                        
                        String measurementUuid = values[0];
                        long   measurementTime = Long.parseLong(values[4]);
                        
                        int detectorId = Integer.parseInt(values[1]);
                        int galaxyId   = Integer.parseInt(values[2]);
                        int personId   = Integer.parseInt(values[3]);
                        
                        double amp1 = Double.parseDouble(values[2]);
                        double amp2 = Double.parseDouble(values[3]);
                        double amp3 = Double.parseDouble(values[4]);
                       
                        
                        Insert insert = table.newInsert();
                        PartialRow insertRow = insert.getRow();
                        
                        
                        boolean isWave = (amp1 > 0.995 && amp3 > 0.995 && amp2 < 0.005);
               
                        insertRow.addString("measurement_id", measurementUuid);
                        insertRow.addLong("measurement_time", measurementTime);
                    
                        insertRow.addInt("detector_id", detectorId);
                        insertRow.addInt("galaxy_id",   galaxyId);
                        insertRow.addInt("person_id",   personId);
                        
                        insertRow.addDouble("amp1", amp1);
                        insertRow.addDouble("amp2", amp2);
                        insertRow.addDouble("amp3", amp3);
                        
                        insertRow.addBoolean("is_wave", isWave);
                        
                        session.apply(insert);
                    }
                    
                    session.flush();
                    
                    session.close();
                    client.shutdown();
                }
            });
            
            return null;
        } 
    });
    
    ssc.start();
    ssc.awaitTermination();
}
```

Upload it using `scp` to an edge node and run the following command to submit the job:

```sh
[ec2-user@ip-172-31-35-239 ~]$ spark-submit gravity-0.1.0.jar \
> ec2-34-212-203-105.us-west-2.compute.amazonaws.com:9092 \
> yeah \
> ip-172-31-37-80.us-west-2.compute.internal:7051 \
> impala::impala_kudu.mars_spark
17/06/30 14:54:10 INFO spark.SparkContext: Running Spark version 1.6.0
;17/06/30 14:54:10 INFO spark.SecurityManager: Changing view acls to: ec2-user
17/06/30 14:54:10 INFO spark.SecurityManager: Changing modify acls to: ec2-user
17/06/30 14:54:10 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(ec2-user); users with modify permissions: Set(ec2-user)
17/06/30 14:54:11 INFO util.Utils: Successfully started service 'sparkDriver' on port 46632.
17/06/30 14:54:11 INFO slf4j.Slf4jLogger: Slf4jLogger started
17/06/30 14:54:11 INFO Remoting: Starting remoting

...
```

---

### <a name="s4"></a> Streaming data from Kafka topic into Kudu using Envelope [^](#top)

#### Envelope config file 

```
application {
    name = Architecture Bootcamp Streaming
    batch.milliseconds = 5000
}

steps {
    traffic {
        input {
            type = kafka
            brokers = "ec2-34-212-203-105.us-west-2.compute.amazonaws.com:9092"
            topics = yeah
            encoding = string
            translator {
                type = delimited
                delimiter = ","
                field.names = [measurement_id,detector_id,galaxy_id,person_id,measurement_time,amp1,amp2,amp3]
                field.types = [string,int,int,int,long,double,double,double]
            }
            window {
                enabled = true
                milliseconds = 60000
            }
        }
    }
    trafficwindow {
        dependencies = [traffic]
        deriver {
            type = sql
            query.literal = """
                SELECT
					*,
					CASE WHEN amp1 > 0.995 AND amp2 < 0.005 AND amp3 > 0.995 
						THEN TRUE
                        ELSE FALSE
                END isawave
                FROM traffic"""
        }
        planner {
            type = upsert
        }
        output {
            type = kudu
            connection = "ec2-52-38-84-189.us-west-2.compute.amazonaws.com:7051"
            table.name = "impala::impala_kudu.mars_spark_enr"
        }
    }
}
```


#### Install and build Envelope project from __Cloudera-labs__  [^](#top)

```
[ec2-user@ip-172-31-47-135 ~]$ wget http://download.nextag.com/apache/maven/maven-3/3.5.0/binaries/apache-maven-3.5.0-bin.tar.gz
[ec2-user@ip-172-31-47-135 ~]$ tar xvf apache-maven-3.5.0-bin.tar.gz
[ec2-user@ip-172-31-47-135 ~]$ sudo mv apache-maven-3.5.0 /var
[ec2-user@ip-172-31-47-135 ~]$ sudo chmod -R 755 /var/apache-maven-3.5.0
[ec2-user@ip-172-31-47-135 ~]$ sudo yum install git
[ec2-user@ip-172-31-47-135 ~]$ mkdir envelope
[ec2-user@ip-172-31-47-135 ~]$ cd envelope
[ec2-user@ip-172-31-47-135 envelope]$ git init
[ec2-user@ip-172-31-47-135 envelope]$ git clone https://github.com/cloudera-labs/envelope.git
[ec2-user@ip-172-31-47-135 envelope]$ /var/apache-maven-3.5.0/bin/mvn clean package
```

#### Run Envelope as Spark applicaiton  [^](#top)

```
[ec2-user@ip-172-31-47-135 envelope]$ 
[ec2-user@ip-172-31-47-135 envelope]$ spark-submit envelope/target/envelope-*.jar kudu2mars.conf
17/06/30 14:37:03 INFO envelope.EnvelopeMain: Envelope application started
17/06/30 14:37:03 INFO envelope.EnvelopeMain: Configuration loaded
17/06/30 14:37:03 INFO run.Runner: Starting getting steps
17/06/30 14:37:03 INFO run.Runner: Adding batch step: trafficwindow
17/06/30 14:37:03 INFO run.Runner: With configuration: Config(SimpleConfigObject({"dependencies":["traffic"],"planner":{"type":"upsert"},"output":{"connection":"ec2-52-38-84-189.us-west-2.compute.amazonaws.com:7051","table":{"name":"impala::impala_kudu.mars_spark_enr"},"type":"kudu"},"deriver":{"query":{"literal":"\n                SELECT \n\t\t*,\n\t\tCASE WHEN amp1 > 0.995 AND amp2 < 0.005 AND amp3 > 0.995 THEN TRUE\n\t\t\tELSE FALSE\n\t\tEND isawave\n\t\tFROM traffic"},"type":"sql"}}))
17/06/30 14:37:03 INFO run.Runner: Adding streaming step: traffic
17/06/30 14:37:03 INFO run.Runner: With configuration: Config(SimpleConfigObject({"input":{"topics":"yeah","window":{"enabled":true,"milliseconds":60000},"translator":{"field":{"names":["measurement_id","detector_id","galaxy_id","person_id","measurement_time","amp1","amp2","amp3"],"types":["string","int","int","int","long","double","double","double"]},"delimiter":",","type":"delimited"},"encoding":"string","type":"kafka","brokers":"ec2-34-212-203-105.us-west-2.compute.amazonaws.com:9092"}}))
17/06/30 14:37:03 INFO run.Runner: Finished getting steps
17/06/30 14:37:03 INFO run.Runner: Steps instatiated
17/06/30 14:37:03 INFO run.Runner: Streaming step(s) identified
17/06/30 14:37:03 INFO run.Runner: Independent steps are:
17/06/30 14:37:03 INFO run.Runner: Started batch for steps:
17/06/30 14:37:03 INFO run.Runner: Finished batch for steps:
17/06/30 14:37:03 INFO run.Runner: Streaming steps are: traffic
17/06/30 14:37:03 INFO run.Runner: Setting up streaming step: traffic
17/06/30 14:37:04 INFO spark.SparkContext: Running Spark version 1.6.0
17/06/30 14:37:04 INFO spark.SecurityManager: Changing view acls to: ec2-user
17/06/30 14:37:04 INFO spark.SecurityManager: Changing modify acls to: ec2-user
17/06/30 14:37:04 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(ec2-user); users with modify permissions: Set(ec2-user)
17/06/30 14:37:04 INFO util.Utils: Successfully started service 'sparkDriver' on port 40781.
17/06/30 14:37:05 INFO slf4j.Slf4jLogger: Slf4jLogger started
17/06/30 14:37:05 INFO Remoting: Starting remoting
17/06/30 14:37:05 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriverActorSystem@172.31.47.135:41082]
17/06/30 14:37:05 INFO Remoting: Remoting now listens on addresses: [akka.tcp://sparkDriverActorSystem@172.31.47.135:41082]
17/06/30 14:37:05 INFO util.Utils: Successfully started service 'sparkDriverActorSystem' on port 41082.
17/06/30 14:37:05 INFO spark.SparkEnv: Registering MapOutputTracker
17/06/30 14:37:05 INFO spark.SparkEnv: Registering BlockManagerMaster
17/06/30 14:37:05 INFO storage.DiskBlockManager: Created local directory at /tmp/blockmgr-c8e09d78-d6a8-4567-8307-664a12b412f5
17/06/30 14:37:05 INFO storage.MemoryStore: MemoryStore started with capacity 530.3 MB
17/06/30 14:37:05 INFO spark.SparkEnv: Registering OutputCommitCoordinator
17/06/30 14:37:05 INFO util.Utils: Successfully started service 'SparkUI' on port 4040.
17/06/30 14:37:05 INFO ui.SparkUI: Started SparkUI at http://172.31.47.135:4040
17/06/30 14:37:05 INFO spark.SparkContext: Added JAR file:/home/ec2-user/envelope/envelope/target/envelope-0.3.0.jar at spark://172.31.47.135:40781/jars/envelope-0.3.0.jar with timestamp 1498847825996
17/06/30 14:37:06 INFO client.RMProxy: Connecting to ResourceManager at ip-172-31-47-135.us-west-2.compute.internal/172.31.47.135:8032
17/06/30 14:37:06 INFO yarn.Client: Requesting a new application from cluster with 3 NodeManagers
17/06/30 14:37:06 INFO yarn.Client: Verifying our application has not requested more than the maximum memory capability of the cluster (5336 MB per container)
17/06/30 14:37:06 INFO yarn.Client: Will allocate AM container, with 896 MB memory including 384 MB overhead
17/06/30 14:37:06 INFO yarn.Client: Setting up container launch context for our AM
17/06/30 14:37:06 INFO yarn.Client: Setting up the launch environment for our AM container
17/06/30 14:37:06 INFO yarn.Client: Preparing resources for our AM container
17/06/30 14:37:07 INFO yarn.Client: Uploading resource file:/tmp/spark-83910d8d-ff7f-40ac-a914-3af4cb023f16/__spark_conf__5280266056101106745.zip -> hdfs://ip-172-31-47-135.us-west-2.compute.internal:8020/user/ec2-user/.sparkStaging/application_1498777759638_0011/__spark_conf__5280266056101106745.zip
17/06/30 14:37:07 INFO spark.SecurityManager: Changing view acls to: ec2-user
17/06/30 14:37:07 INFO spark.SecurityManager: Changing modify acls to: ec2-user
17/06/30 14:37:07 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(ec2-user); users with modify permissions: Set(ec2-user)
17/06/30 14:37:07 INFO yarn.Client: Submitting application 11 to ResourceManager
17/06/30 14:37:07 INFO impl.YarnClientImpl: Submitted application application_1498777759638_0011
17/06/30 14:37:08 INFO yarn.Client: Application report for application_1498777759638_0011 (state: ACCEPTED)
17/06/30 14:37:08 INFO yarn.Client:
         client token: N/A
         diagnostics: N/A
         ApplicationMaster host: N/A
         ApplicationMaster RPC port: -1
         queue: root.users.ec2-user
         start time: 1498847827541
         final status: UNDEFINED
         tracking URL: http://ip-172-31-47-135.us-west-2.compute.internal:8088/proxy/application_1498777759638_0011/
         user: ec2-user
17/06/30 14:37:09 INFO yarn.Client: Application report for application_1498777759638_0011 (state: ACCEPTED)
. . . . 
 . . . . 
17/06/30 14:37:17 INFO kafka.DirectKafkaInputDStream: Slide time = 5000 ms
17/06/30 14:37:17 INFO kafka.DirectKafkaInputDStream: Storage level = StorageLevel(false, false, false, false, 1)
17/06/30 14:37:17 INFO kafka.DirectKafkaInputDStream: Checkpoint interval = null
17/06/30 14:37:17 INFO kafka.DirectKafkaInputDStream: Remember duration = 65000 ms
17/06/30 14:37:17 INFO kafka.DirectKafkaInputDStream: Initialized and validated org.apache.spark.streaming.kafka.DirectKafkaInputDStream@7288cea1
17/06/30 14:37:17 INFO dstream.FlatMappedDStream: Slide time = 5000 ms
17/06/30 14:37:17 INFO dstream.FlatMappedDStream: Storage level = StorageLevel(false, true, false, false, 1)
17/06/30 14:37:17 INFO dstream.FlatMappedDStream: Checkpoint interval = null
17/06/30 14:37:17 INFO dstream.FlatMappedDStream: Remember duration = 65000 ms
17/06/30 14:37:17 INFO dstream.FlatMappedDStream: Initialized and validated org.apache.spark.streaming.dstream.FlatMappedDStream@13d89e6c
17/06/30 14:37:17 INFO dstream.WindowedDStream: Slide time = 5000 ms
17/06/30 14:37:17 INFO dstream.WindowedDStream: Storage level = StorageLevel(false, false, false, false, 1)
17/06/30 14:37:17 INFO dstream.WindowedDStream: Checkpoint interval = null
17/06/30 14:37:17 INFO dstream.WindowedDStream: Remember duration = 5000 ms
17/06/30 14:37:17 INFO dstream.WindowedDStream: Initialized and validated org.apache.spark.streaming.dstream.WindowedDStream@56231e00
17/06/30 14:37:17 INFO dstream.ForEachDStream: Slide time = 5000 ms
17/06/30 14:37:17 INFO dstream.ForEachDStream: Storage level = StorageLevel(false, false, false, false, 1)
17/06/30 14:37:17 INFO dstream.ForEachDStream: Checkpoint interval = null
17/06/30 14:37:17 INFO dstream.ForEachDStream: Remember duration = 5000 ms
17/06/30 14:37:17 INFO dstream.ForEachDStream: Initialized and validated org.apache.spark.streaming.dstream.ForEachDStream@1b86be2e
17/06/30 14:37:17 INFO util.RecurringTimer: Started timer for JobGenerator at time 1498847840000
17/06/30 14:37:17 INFO scheduler.JobGenerator: Started JobGenerator at 1498847840000 ms
17/06/30 14:37:17 INFO scheduler.JobScheduler: Started JobScheduler
17/06/30 14:37:17 INFO streaming.StreamingContext: StreamingContext started
17/06/30 14:37:17 INFO run.Runner: Streaming context started
17/06/30 14:37:20 INFO dstream.FlatMappedDStream: Slicing from 1498847785000 ms to 1498847840000 ms (aligned to 1498847785000 ms and 1498847840000 ms)
17/06/30 14:37:20 INFO dstream.FlatMappedDStream: Time 1498847835000 ms is invalid as zeroTime is 1498847835000 ms and slideDuration is 5000 ms and difference is 0 ms
17/06/30 14:37:20 INFO utils.VerifiableProperties: Verifying properties
17/06/30 14:37:20 INFO utils.VerifiableProperties: Property group.id is overridden to
17/06/30 14:37:20 INFO utils.VerifiableProperties: Property zookeeper.connect is overridden to
17/06/30 14:37:20 INFO scheduler.JobScheduler: Added jobs for time 1498847840000 ms
17/06/30 14:37:20 INFO scheduler.JobScheduler: Starting job streaming job 1498847840000 ms.0 from job set of time 1498847840000 ms
17/06/30 14:37:20 INFO run.Runner: Immediate dependent steps of traffic are: trafficwindow


```
