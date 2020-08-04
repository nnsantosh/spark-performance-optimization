# Spark Joins

  ## Shuffle Sort Join

    When large to large dataset is joined then shuffle sort join is used

  ## BroadCastHash Join

    When large to small dataset is joined then spark implicitly uses broadcast hash join
    If we are aware about our dataset then we can force spark to use broadcast join. The smaller dataset can easily fit in one executor or driver.
    In this case the smaller dataset is broadcasted to all the executors holding larger dataset and join is performed without a shuffle.
    Example:
    val joinExpr = flightTimeDf1.col(colName="id") === flightTimeDf2.col(colName="id")
    val joinDF = flightTimeDf1.join(broadcast(flightTimeDf2), joinExpr, joinType="inner")


  ## Improve performance during Joins
  
    1. Reduce the shuffles

    2. Find out the number of unique keys for your join condition and let the number of shuffle partitions equal to the unique keys
        spark.conf.set("spark.sql.shuffle.partitions",<number of unique keys>)
        Ideally
        No of shuffle partitions = No of unique keys
        This will ensure data is distributed across partitions and down the line there will be less shuffle
        No of shuffle partitions  = No of executors
        Then this will ensure maximum parallelism. If there are more number of executors than your shuffle partitions/unique keys then lot of executors will go unused.

    3. Filter out unnecessary data and reduce your dataset size before performing join.

    4. Avoid skewed data where one unique key has more data than others. This will result in the task performing on the larger partition taking
        more time than other tasks.
        Example: Fastest moving product partition vs slowest moving product partition
        In this case you need to apply some hack to distribute your data across partitions uniformly. This may not be straight forward.
        You may have to use cardinal keys.

    5. Use Bucket Joins
        By bucketing your dataset you can avoid shuffle altogether at the join time.
        The number of buckets or the number of partitions is a critical decision. This number can be decided only if we understand our dataset.
        We should also know something about the available cluster capacity.
        Maximize parallelism and minimize skew.

        Example:
        val flightTimeDf1 = spark.read.json(path = "data/d1/")
        val flightTimeDf2 = spark.read.json(path = "data/d2/")

        spark.sql("CREATE DATABASE IF NOT EXISTS MY_DB")
        spark.sql("USE MY_DB")
        flightTimeDf1.coalesce(numPartitions=1)
          .write.bucketBy(numBuckets = 3, colName = "id")
          .mode(SaveMode.Overwrite)
          .saveAsTable(tableName="MY_DB.flight_data1")
        flightTimeDf2.coalesce(numPartitions=1)
          .write.bucketBy(numBuckets = 3, colName = "id")
          .mode(SaveMode.Overwrite)
          .saveAsTable(tableName="MY_DB.flight_data2")

        val df3 = spark.read.table(tableName = "MY_DB.flight_data1")
        val df4 = spark.read.table(tableName = "MY_DB.flight_data2")

        //To avoid broadcast join being applied by spark implicitly
        spark.conf.set("spark.sql.autoBroadcastJoinThreshold",-1)

        val joinExpr = df3.col(colName="id") === df4.col(colName="id")
        val joinDF = df3.join(df4, joinExpr, joinType="inner")
