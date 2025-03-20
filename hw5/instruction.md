```markdown
# Инструкция по запуску ETL-процесса с помощью Prefect


## 1. Подключение к серверу и проброс портов
```bash
ssh -L 9870:127.0.0.1:9870 \
    -L 8088:127.0.0.1:8088 \
    -L 19888:127.0.0.1:19888 \
    team@ip_jn
```

---

## 2. Установка системных зависимостей
```bash
sudo apt update
sudo apt install -y python3-venv python3-pip
```

---

## 3. Переключение на пользователя `hadoop`
```bash
sudo –i –u hadoop
```

---

## 4. Установка Spark 3.5.3
```bash
wget https://archive.apache.org/dist/spark/spark-3.5.3/spark-3.5.3-bin-hadoop3.tgz
tar –xzvf spark-3.5.3-bin-hadoop3.tgz
```

---

## 5. Настройка переменных окружения
Добавьте в `~/.bashrc` или выполните команды:
```bash
export HADOOP_CONF_DIR="/home/hadoop/hadoop-3.4.0/etc/hadoop"
export HIVE_HOME="/home/hadoop/apache-hive-4.0.2-bin"
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
export SPARK_LOCAL_IP="ip_jn"  # Замените на локальный IP сервера
export SPARK_DIST_CLASS_PATH="/home/hadoop/spark-3.5.3-bin-hadoop3/jars/*:/home/hadoop/hadoop-3.4.0/etc/hadoop:/home/hadoop/hadoop-3.4.0/share/hadoop/common/lib/*:/home/hadoop/hadoop-3.4.0/share/hadoop/common/*:/home/hadoop/hadoop-3.4.0/share/hadoop/hdfs:/home/hadoop/hadoop-3.4.0/share/hadoop/hdfs/lib/*:/home/hadoop/hadoop-3.4.0/share/hadoop/hdfs/*:/home/hadoop/hadoop-3.4.0/share/hadoop/mapreduce/*:/home/hadoop/hadoop-3.4.0/share/hadoop/yarn:/home/hadoop/hadoop-3.4.0/share/hadoop/yarn/lib/*:/home/hadoop/hadoop-3.4.0/share/hadoop/yarn/*:/home/hadoop/apache-hive-4.0.0-alpha-2-bin/lib/*"

cd spark-3.5.3-bin-hadoop3/
export SPARK_HOME=`pwd`
export PATH=$SPARK_HOME/bin:$PATH
export PYTHONPATH=$(ZIPS=("$SPARK_HOME"/python/lib/*.zip); IFS=:; echo "${ZIPS[*]}")$PYTHONPATH
cd ~
```

---

## 6. Создание виртуального окружения и установка Python-пакетов
```bash
python3 -m venv venv
source venv/bin/activate
pip install –U pip
pip install ipython onetl[files] prefect
```

---

## 7. Создание и запуск ETL-скрипта
Создайте файл `Dag.py`:
```bash
nano Dag.py
```

Вставьте содержимое:
```python
from onetl.db import DBWriter
from onetl.file.format import CSV
from onetl.connection import Hive
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from prefect import flow, task

@task
def get_spark():
    spark = SparkSession.builder \
        .master("yarn") \
        .appName("spark-with-yarn") \
        .config("spark.sql.warehouse.dir", "/user/hive/warehouse") \
        .config("spark.hive.metastore.uris", "thrift://team-8-jn:9083") \
        .enableHiveSupport() \
        .getOrCreate()
    return spark

@task
def stop_spark(spark):
    spark.stop()

@task
def extract(spark):
    hdfs = SparkHDFS(host="team-8-nn", port=9000, spark=spark, cluster="test")
    reader = FileDFReader(connection=hdfs, format=CSV(delimiter=",", header=True), source_path="/input")
    df = reader.run(['titanic.csv'])
    return df

@task
def transform(df):
    df = df.withColumn("reg_year", F.col("Name").substr(0, 4))
    return df

@task
def load(spark, df):
    hive = Hive(spark=spark, cluster="test")
    writer = DBWriter(connection=hive, table="test.hive_parts", options={"if_exists": "replace_entire_table", "partitionBy": "reg_year"})
    writer.run(df)

@flow
def main_flow():
    spark = get_spark()
    try:
        df = extract(spark)
        df_transformed = transform(df)
        load(spark, df_transformed)
    finally:
        stop_spark(spark)

if __name__ == "__main__":
    main_flow()
```

---

## 8. Запуск ETL-процесса
```bash
python Dag.py
```

---
