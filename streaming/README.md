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
