## <center>Exercise Objectives</center>

### <center>Ingest measurements and reference data from Oracle </center>

    Tables: measurements, detectors, galaxies, astrophysicists

    * 500 million measurements
    
    * 8 detectors, 128 galaxies, 106 astrophysicists


### <center>Make the tables available to Impala for querying in Hue</center>


### <center>Additional functionality ideas </center>

* More import parallelism

* Write directly to Parquet

* Compression

* Ingest straight into a partitioned table

* First, create views to simplify the presentation data model

   - Detect gravitational waves for the user

   - Pre-join reference tables

* Next create additional tables to speed up the presentation data model

   - Physicalize the views

   - Provide pre-aggregated results

   - Convert DOUBLEs to DECIMALs

