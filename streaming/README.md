# mars-june-2017
Mission to Mars


# Exercise

1. Run measurements stream using the provided data generator

2. Build a `generator -> Flume -> HBase` pipeline
    * Then switch out HBase for HDFS

3. Build a `generator -> Flume -> Kafka -> Spark Streaming -> HBase` pipeline
    * Then switch out HBase for Kudu

4. First use built-in Spark-Kudu API, then use Envelope
    * Then switch out Kudu for Solr



## Setting up a pseudo data generator source

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
