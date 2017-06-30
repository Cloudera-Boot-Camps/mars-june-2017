## <center>Batch Data Processing in EDH</center> 


__Create Oozie Workflow To Do the Following__

 * Import some of the measurements from Oracle (‘sqoop’ action type)

 * Import all of the reference data from Oracle (‘sqoop’ action type)
    
 * Transform the data (‘hive2’ action type)
 
   - Join all the tables
 
   - Save as Parquet
  
   - Correct data types
   
   - Add a flag attribute for gravitational waves

__Optimize the Workflow__

 * Use subworkflows for Sqoop imports

 * Import the Oracle tables in parallel

 * Add a coordinator that calls the workflow on a schedule

 * Run Hive-on-Spark instead of Hive-on-MapReduce

 * Build a Spark SQL project to run the transformations instead of Hive

---

## Oozie Workflow Created
