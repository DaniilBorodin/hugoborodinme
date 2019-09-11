+++
author = "Daniil Borodin"
cover = ""
date = "2019-09-11"
description = ""
keywords = ["", ""]
showFullContent = false
tags = [""]
title = "Чтение и проверка parquet файлов в HDFS c использованием Java"

+++
Работая с HADOOP , тестировщику часто приходится валидировать полученные итоговые parquet файлы. Опишу как это делал я в автотестах.

Для начала несколько удобных команд HDFS File System Shell.

Посмотреть содержимое папки в HDFS:

```bash
hdfs dfs -ls /path/to/parquets/
```

Прочитать сожержимое parquet файла:

```bash
hdfs dfs -cat /path/to/parquet/part-00000-2e2c232a-a50c-4885-aba0-53d10bb47b75-c000.snappy.parquet
```

Помимо этих команд есть множество других полезных -cp , -copyToLocal, -stat и т.д. Подробнее можно посмотреть [тут](https://hadoop.apache.org/docs/r2.4.1/hadoop-project-dist/hadoop-common/FileSystemShell.html).

Также часто бывает полезен [parquet-tools](http://central.maven.org/maven2/org/apache/parquet/parquet-tools/1.9.0/).

```bash
 hadoop jar ~/parquet-tools-1.9.0.jar
```

Умеет показывать содержимое parquet , а также может делать merge.

В автотестах читаем и валидируем parquet используя [Apache Spark](https://spark.apache.org/). Для этого в pom файл нашего проекта на Maven добавим следующие:

```xml
</dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>2.3.0</version>
        </dependency>
```

И библиотеку spark-sql:

```xml
<!-- https://mvnrepository.com/artifact/org.apache.spark/spark-sql -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>2.3.0</version>
        </dependency>
```

В файл \\env\\default\\tests.properties добавим адрес нашей HADOOP master node и путь к каталогу моделей.

```bash
HADOOP_MASTER_NODE=127.0.0.1
MODELS_PATH=me.borodin.qa.parquet.models.
```

Вместо 127.0.0.1 ставим IP HADOOP ноды с которой собираетесь работать. Модель

Длаее пишем класс констант для дальнейшего использования в нашем проекте.

```java
package me.borodin.qa.parquet;

public class Constants {

    public static String HADOOP_MASTER_NODE = System.getenv("HADOOP_MASTER_NODE");
}
```
Допустим у нас есть parquet файл со следующей схемой:

```bash
[d_borodin@borodinme ~]$ hadoop jar ~/parquet-tools-1.9.0.jar schema /part-00000.snappy.parquet
message spark_schema { 
  required int32 companyID;
  required binary companyName (UTF8);
  optional binary companyDesc (UTF8);
}
```
В models добаваляем POJO для данного parquet файла. 

```java
package me.borodin.qa.parquet.models;

import lombok.Data;

@Data

public class Company {

    private Integer companyID;
    private String companyName;
    private String companyDesc;   
}
```

Затем напишем простой класс SparkManager который нам будет возвращать объект SparkSession для дальнейшей работы:

```java
package me.borodin.qa.parquet;

import org.apache.spark.SparkConf;
import org.apache.spark.sql.SparkSession;

public class SparkManager {

    public SparkSession startSparkSession() {
        return createSparkSession();
    }

    public void closeSparkSession(SparkSession session) {
        session.close();
    }

    private SparkSession createSparkSession() {
        SparkConf sparkConf = new SparkConf()
                .setMaster("local[*]")
                .set("spark.executor.memory", "1G")
                .set("spark.driver.memory", "1G")
                .set("spark.sql.shuffle.partitions", "1")
                .setAppName("myappname");

        return SparkSession.builder().config(sparkConf).getOrCreate();
    }
}
```

Далее пишем класс в котором непосредственно будут методы чтения parquet файлов:

```java

    public void readParquetFromRemoteHdfs(String filePath, String dataModel) {
    
        String pathToParquet = "hdfs://" + Constants.HADOOP_MASTER_NODE + filePath;
        String modelClassName = Constants.MODELS_PATH + dataModel;
        Map<String, Dataset> parquetReaults = new HashMap<String, Dataset>();

        Dataset parquetDataset = sparkSession
                .read()
                .parquet(pathToParquet)
                .as(Encoders.bean(Class.forName(modelClassName)));
        parquetDataset.printSchema();

        parquetReaults.put(dataModel, parquetDataset);
        logger.info(String.format("Parquet-файл %s прочитан и соответствует модели %s", filePath, dataModel));
    }
}
```