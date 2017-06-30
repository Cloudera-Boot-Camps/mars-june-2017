# mars-june-2017
Mission to Mars


# Exercise

1. [Run measurements stream using the provided data generator](#s1)

2. [Build a `generator -> Flume -> HBase` pipeline](#s2)
    * Then switch out HBase for HDFS

3. [Build a `generator -> Flume -> Kafka -> Spark Streaming -> HBase` pipeline](#s3)
    * Then switch out HBase for Kudu

4. [First use built-in Spark-Kudu API, then use Envelope](#s4)
    * Then switch out Kudu for Solr



## <a name="s1"></a> Setting up a pseudo data generator source

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

## Creating a HBase table

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

## Setting up Flume source to listen on TCP socket

* Debugging tips.

