### DataFrame
+ DataFrame consists of a series of records *(like rows in a table)*, that are of type Row,
  and a number of columns *(like columns in a spreadsheet)*
+ Schemas define the name as well as the
   type of data in each column
+  Partitioning of the DataFrame defines the layout of the DataFrame or
   Dataset’s physical distribution across the cluster
+  The partitioning scheme defines how that is
   allocated
   
###### Load dataFrame
> ```scala
> val df = spark.read.format("json").load("/data/flight-data/json/2015-summary.json")
> df.printSchema() //print's schema
> ```

### Schemas
+ A schema defines the column names and types of a DataFrame. We can either let a data source define
  the schema (called schema-on-read) or we can define it explicitly ourselves
+ A schema is a StructType made up of a number of fields, StructFields, that have a name, type, a
  Boolean flag which specifies whether that column can contain missing or null values

> ```scala
> import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
> import org.apache.spark.sql.types.Metadata
> val myManualSchema = StructType(Array(
> StructField("DEST_COUNTRY_NAME", StringType, true),
> StructField("ORIGIN_COUNTRY_NAME", StringType, true),
> StructField("count", LongType, false,
> Metadata.fromJson("{\"hello\":\"world\"}"))
> ))
> val df = spark.read.format("json").schema(myManualSchema)
> .load("/data/flight-data/json/2015-summary.json")
> ```  

### Columns and Expressions
###### Cloumns:
+ Columns in Spark are similar to columns in a spreadsheet, R dataframe, or pandas DataFrame. You
  can select, manipulate, and remove columns from DataFrames and these operations are represented as
  *expressions*
>```scala
>import org.apache.spark.sql.functions.{col, column}
>col("someColumnName")
>column("someColumnName")
>// the below are scala implicit shorthand ways
>$"myColumn"
>'myColumn
>//If you need to refer to a specific DataFrame’s column, you can use the col method on the specific
>    DataFrame. This can be useful when you are performing a join and need to refer to a specific column
>    in one DataFrame that might share a name with another column in the joined DataFrame.
> df.col("count")
>```
*The $ allows us to designate a string as a special string that should refer to an expression. The tick mark (') is a special
 thing called a symbol; this is a Scala-specific construct of referring to some identifier*
 
###### Expressions:
+ columns are *Expressions*, An expression is a set of transformations on one or more values in a record in a DataFrame
+ an expression, created via the expr function, is just a DataFrame column reference. In the simplest case, *expr("someCol")* is equivalent to *col("someCol")*
+ *expr("someCol - 5")* is the same transformation as performing *col("someCol") - 5*, or even
  *expr("someCol") - 5*. That’s because Spark compiles these to a logical tree specifying the order of operations.
> ```scala
> import org.apache.spark.sql.functions.expr
> expr("(((someCol + 5) * 200) - 6) < otherCol")
> //the below is logical tree for this expression
> ```

![](https://github.com/anirudhreddy92/cca175scala/blob/master/logical%20tree.PNG)

>```scala
>//way to access columns in dataframe
>spark.read.format("json").load("/data/flight-data/json/2015-summary.json")
> .columns
>```

### Records and Rows
+ Each row in a DataFrame is a single record. Spark represents this record as an object of type Row.
+ Spark manipulates Row objects using column expressions in order to produce usable values
+ You can create rows by manually instantiating a Row object with the values that belong in each column.
>```scala
>import org.apache.spark.sql.Row
>val myRow = Row("Hello", null, 1, false)
>myRow(0) // type Any
>myRow(0).asInstanceOf[String] // String
>myRow.getString(0) // String
>myRow.getInt(2) // Int
>```

### Creating DataFrames
###### *From data source:*
>```scala
>val df = spark.read.format("json")
>.load("/data/flight-data/json/2015-summary.json")
>df.createOrReplaceTempView("dfTable")
>```

###### *On the fly:*
>```scala
>import org.apache.spark.sql.Row
>import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
>val myManualSchema = new StructType(Array(
>new StructField("some", StringType, true),
>new StructField("col", StringType, true),
>new StructField("names", LongType, false)))
>val myRows = Seq(Row("Hello", null, 1L))
>val myRDD = spark.sparkContext.parallelize(myRows)
>val myDf = spark.createDataFrame(myRDD, myManualSchema)
>myDf.show()
>```                          

###### select and selectExpr
+ select and selectExpr allow you to do the DataFrame equivalent of SQL queries on a table of data
+ you can refer to columns in a number of different ways

>```scala
>import org.apache.spark.sql.functions.{expr, col, column}
>df.select(
>df.col("DEST_COUNTRY_NAME"),
>col("DEST_COUNTRY_NAME"),
>column("DEST_COUNTRY_NAME"),
>'DEST_COUNTRY_NAME,
>$"DEST_COUNTRY_NAME",
>expr("DEST_COUNTRY_NAME"))
>.show(2)
>```
+ changing column names using AS and alias
>```scala
>df.select(expr("DEST_COUNTRY_NAME AS destination")).show(2)
>df.select(expr("DEST_COUNTRY_NAME as destination").alias("DEST_COUNTRY_NAME")).show(2)
>  //using selectExpr
>df.selectExpr("DEST_COUNTRY_NAME as newColumnName", "DEST_COUNTRY_NAME").show(2)
>```

+ we can build comples expressions that create new DataFrame using selectExpr wit non-aggregating SQL statemet
>```scala
>df.selectExpr(
>"*", // include all original columns
> "(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry")
>.show(2)
>```

>+-----------------+-------------------+-----+-------------+
>|DEST_COUNTRY_NAME|ORIGIN_COUNTRY_NAME|count|withinCountry|
>+-----------------+-------------------+-----+-------------+
>| United States| Romania| 15| false|
>| United States| Croatia| 1| false|
>+-----------------+-------------------+-----+-------------+

+ aggregations using selectExprn
>`df.selectExpr("avg(count)", "count(distinct(DEST_COUNTRY_NAME))").show(2)`
 
 +-----------+---------------------------------+
 | avg(count)|count(DISTINCT DEST_COUNTRY_NAME)|
 +-----------+---------------------------------+
 |1770.765625| 132|
 +-----------+---------------------------------+
 
###### Converting to Spark Types (Literals)
+ Sometimes, we need to pass explicit values into Spark that are just a value (rather than a new
  column). This might be a constant value or something we’ll need to compare to later on. The way we
  do this is through literals. This is basically a translation from a given programming language’s literal
  value to one that Spark understands. Literals are expressions and you can use them in the same way:
 
 >```scala
 >import org.apache.spark.sql.functions.lit
 >df.select(expr("*"), lit(1).as("One")).show(2)
 >```
+-----------------+-------------------+-----+---+
|DEST_COUNTRY_NAME|ORIGIN_COUNTRY_NAME|count|One|
+-----------------+-------------------+-----+---+
| United States| Romania| 15| 1|
| United States| Croatia| 1| 1|
+-----------------+-------------------+-----+---+

###### Adding Columns
+ we can add column using withColumn
>```scala
>df.withColumn("numberOne", lit(1)).show(2)
>df.withColumn("withinCountry", expr("ORIGIN_COUNTRY_NAME == DEST_COUNTRY_NAME"))
>.show(2)
>```

###### Renaming Columns
>`df.withColumnRenamed("DEST_COUNTRY_NAME", "dest").columns`

###### Removing Columns
>`dfWithLongColName.drop("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")`

###### Changing a Column’s Type (cast)
>`df.withColumn("count2", col("count").cast("long"))`

###### Filtering Rows
>`df.filter(col("count") < 2).show(2)`
>`df.where("count < 2").show(2)`

###### Getting Unique Rows
>`df.select("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME").distinct().count()`

###### Random Samples & Random Splits
>```scala
>//Samplling
>val seed = 5
>val withReplacement = false
>val fraction = 0.5
>df.sample(withReplacement, fraction, seed).count()
>//Split
>val dataFrames = df.randomSplit(Array(0.25, 0.75), seed)
>dataFrames(0).count() > dataFrames(1).count() // False
>```

##### Concatenating and Appending Rows (Union)
> TODO

###### Sorting Rows
>```scala
>df.sort("count").show(5)
>df.orderBy("count", "DEST_COUNTRY_NAME").show(5)
>df.orderBy(col("count"), col("DEST_COUNTRY_NAME")).show(5)
>//Specify sort direction
>import org.apache.spark.sql.functions.{desc, asc}
>df.orderBy(expr("count desc")).show(2)
>df.orderBy(desc("count"), asc("DEST_COUNTRY_NAME")).show(2)
>```
+ *An advanced tip is to use asc_nulls_first, desc_nulls_first, asc_nulls_last, or
 desc_nulls_last to specify where you would like your null values to appear in an ordered
 DataFrame*/

+ *For optimization purposes, it’s sometimes advisable to sort within each partition before another set of
  transformations. You can use the sortWithinPartitions method to do this:*
 >```scala
 >spark.read.format("json").load("/data/flight-data/json/*-summary.json")
 >.sortWithinPartitions("count")
 >```
 
 ###### Repartition and Coalesce
 >TODO
 
 
 



