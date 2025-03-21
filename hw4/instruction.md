

## 0. Работа с данными

### Загрузка данных в HDFS
data.csv должен быть загружен на hdfs предварительно 
На локальной машине выполнить 
```bash
scp data.csv team@jump_host:/tmp/
```
От пользователя hadoop на Jump Node выполнить
```bash
hdfs dfs -put /tmp/data.csv /input
```
---

## 1. Подготовка окружения

### Подключение к кластеру
```bash
ssh -L 9870:127.0.0.1:9870 \
    -L 8088:127.0.0.1:8088 \
    -L 19888:127.0.0.1:19888 \
    team@ip_jn
```

### Установка зависимостей
```bash
sudo apt update
sudo apt install -y python3-venv python3-pip
sudo -i -u hadoop
```

---

## 2. Установка Apache Spark

```bash
wget https://archive.apache.org/dist/spark/spark-3.5.3/spark-3.5.3-bin-hadoop3.tgz
tar -xzvf spark-3.5.3-bin-hadoop3.tgz
```

---

## 3. Настройка переменных окружения

Добавить в `~/.bashrc`:
```bash
export HADOOP_CONF_DIR="/home/hadoop/hadoop-3.4.0/etc/hadoop"
export HIVE_HOME="/home/hadoop/apache-hive-4.0.2-bin"
export HIVE_CONF_DIR=$HIVE_HOME/conf
export SPARK_HOME="/home/hadoop/spark-3.5.3-bin-hadoop3"
export PATH=$PATH:$SPARK_HOME/bin:$HIVE_HOME/bin
export SPARK_LOCAL_IP="<локальный_IP_jn>"
export SPARK_DIST_CLASS_PATH="<пути_из_шага_13_инструкции>"
```

Применить настройки:
```bash
source ~/.bashrc
```

---

## 4. Настройка Python-окружения

```bash
python3 -m venv venv
source venv/bin/activate
pip install -U pip
pip install ipython onetl[files]
```

---

## 5. Конфигурация Spark Session

В окне ipython:
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("yarn") \
    .appName("spark-with-yarn") \
    .config("spark.sql.warehouse.dir", "hdfs://team-8-nn:9000/user/hive/warehouse") \
    .config("hive.metastore.uris", "thrift://team-8-jn:9083") \
    .enableHiveSupport() \
    .getOrCreate()
```

---


### Пример обработки данных в Spark
```python
from pyspark.sql import functions as F
from onetl.connection import SparkHDFS, Hive
from onetl.file import FileDFReader
from onetl.file.format import CSV
from onetl.db import DBWriter

# Инициализация подключений
hdfs = SparkHDFS(
    host="team-8-nn", 
    port=9000, 
    spark=spark,
    cluster="test"
)

hive = Hive(
    spark=spark,
    cluster="test"
)

# Чтение данных
reader = FileDFReader(
    connection=hdfs,
    format=CSV(delimiter=",", header=True),
    source_path="/input"
)

df = reader.run(["data.csv"])
print(f"Total records: {df.count()}")

# Запись в Hive
writer = DBWriter(
    connection=hive,
    table="test.spark_partitions",
    options={"if_exists": "replace_entire_table"}
)

writer.run(df)
```

### Пример партиционирование данных при сохранении
```python
transformed_df = df \
    .withColumn("SurvivedLabel", F.when(F.col("Survived") == 1, "Yes").otherwise("No")) \
    .groupBy("Pclass", "Sex") \
    .agg(
        F.count("*").alias("TotalPassengers"),
        F.round(F.avg("Age"), 2).alias("AvgAge"),
        F.sum("Survived").alias("Survivors")
    ) \
    .filter(F.col("AvgAge") > 18) \
    .orderBy("Pclass", ascending=False)

transformed_df.show(5)

transformed_df.write \
    .partitionBy("Pclass") \
    .mode("overwrite") \
    .option("path", "hdfs://team-8-nn:9000/user/hive/warehouse/spark_partitions") \
    .saveAsTable("test.spark_partitions")
```
---
