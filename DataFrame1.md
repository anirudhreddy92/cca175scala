### Working with Different Types of Data
+ Booleans
 + Numbers
 + Strings
 + Dates and timestamps
 + Handling null
 + Complex types
 + User-defined functions
 
###### read DataFrame
>```scala
>val df = spark.read.format("csv")
>.option("header", "true")
>.option("inferSchema", "true")
>.load("/data/retail-data/by-day/2010-12-01.csv")
>df.printSchema()
>df.createOrReplaceTempView("dfTable")
>```

root<br />
|-- InvoiceNo: string (nullable = true)<br />
|-- StockCode: string (nullable = true)<br />
|-- Description: string (nullable = true)<br />
|-- Quantity: integer (nullable = true)<br />
|-- InvoiceDate: timestamp (nullable = true)<br />
|-- UnitPrice: double (nullable = true)<br />
|-- CustomerID: double (nullable = true)<br />
|-- Country: string (nullable = true)<br />
+---------+---------+--------------------+--------+-------------------+----...<br />
|InvoiceNo|StockCode| Description|Quantity| InvoiceDate|Unit...<br />
+---------+---------+--------------------+--------+-------------------+----...<br />
| 536365| 85123A|WHITE HANGING HEA...| 6|2010-12-01 08:26:00| ...<br />
| 536365| 71053| WHITE METAL LANTERN| 6|2010-12-01 08:26:00| ...<br />
...<br />
| 536367| 21755|LOVE BUILDING BLO...| 3|2010-12-01 08:34:00| ...<br />
| 536367| 21777|RECIPE BOX WITH M...| 4|2010-12-01 08:34:00| ...<br />
+---------+---------+--------------------+--------+-------------------+----...<br />

###### Working with Booleans
+ Scala has some particular semantics regarding the use of == and ===. In Spark, if you want to filter by equality you should
 use === (equal) or =!= (not equal). You can also use the not function and the equalTo method.

>```sccala
>import org.apache.spark.sql.functions.col
>df.where(col("InvoiceNo").equalTo(536365))
>.select("InvoiceNo", "Description")
>.show(5, false)
>//
>df.where(col("InvoiceNo") === 536365)
>.select("InvoiceNo", "Description")
>.show(5, false)
>//
>df.where("InvoiceNo = 536365")
>.show(5, false)
>df.where("InvoiceNo <> 536365")
>.show(5, false)
>//
>val priceFilter = col("UnitPrice") > 600
>val descripFilter = col("Description").contains("POSTAGE")
>df.where(col("StockCode").isin("DOT")).where(priceFilter.or(descripFilter))
>.show()
>```

+---------+---------+--------------+--------+-------------------+---------+...<br />
|InvoiceNo|StockCode| Description|Quantity| InvoiceDate|UnitPrice|...<br />
+---------+---------+--------------+--------+-------------------+---------+...<br />
| 536544| DOT|DOTCOM POSTAGE| 1|2010-12-01 14:32:00| 569.77|...<br />
| 536592| DOT|DOTCOM POSTAGE| 1|2010-12-01 17:06:00| 607.49|...<br />
+---------+---------+--------------+--------+-------------------+---------+...<br />

+ Boolean expressions are not just reserved to filters. To filter a DataFrame, you can also just specify a Boolean column
>```scala
>val DOTCodeFilter = col("StockCode") === "DOT"
>val priceFilter = col("UnitPrice") > 600
>val descripFilter = col("Description").contains("POSTAGE")
>df.withColumn("isExpensive", DOTCodeFilter.and(priceFilter.or(descripFilter)))
>.where("isExpensive")
>.select("unitPrice", "isExpensive").show(5)
>//
>import org.apache.spark.sql.functions.{expr, not, col}
 >df.withColumn("isExpensive", not(col("UnitPrice").leq(250)))
 >.filter("isExpensive")
 >.select("Description", "UnitPrice").show(5)
 >df.withColumn("isExpensive", expr("NOT UnitPrice <= 250"))
 >.filter("isExpensive")
 >.select("Description", "UnitPrice").show(5)
>```

+ *if you’re working with null data when creating Boolean expressions. If there is a null in
    your data, you’ll need to treat things a bit differently. Here’s how you can ensure that you perform a null-safe equivalence
    test*
>`df.where(col("Description").eqNullSafe("hello")).show()`

###### Working with Numbers
>```scala
>import org.apache.spark.sql.functions.{expr, pow}
>val fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5
>df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)
>//in Sqpark SQL
>df.selectExpr(
>"CustomerId",
>"(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)
>```
+----------+------------------+<br />
|CustomerId| realQuantity|<br />
+----------+------------------+<br />
| 17850.0|239.08999999999997|<br />
| 17850.0| 418.7156|<br />
+----------+------------------+<br />

+ Rounding 
>```scala
>import org.apache.spark.sql.functions.{round, bround}
>df.select(round(col("UnitPrice"), 1).alias("rounded"), col("UnitPrice")).show(5)
>```

+ By default, the round function rounds up if you’re exactly in between two numbers. You can round
  down by using the bround
>```scala
>import org.apache.spark.sql.functions.lit
>df.select(round(lit("2.5")), bround(lit("2.5"))).show(2)
>```

+ summary statistics
>`df.describe().show()`

+-------+------------------+------------------+------------------+<br />
|summary| Quantity| UnitPrice| CustomerID|<br />
+-------+------------------+------------------+------------------+<br />
| count| 3108| 3108| 1968|<br />
| mean| 8.627413127413128| 4.151946589446603|15661.388719512195|<br />
| stddev|26.371821677029203|15.638659854603892|1854.4496996893627|<br />
| min| -24| 0.0| 12431.0|<br />
| max| 600| 607.49| 18229.0|<br />
+-------+------------------+------------------+------------------+<br />





