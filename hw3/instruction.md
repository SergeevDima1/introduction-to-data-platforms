# Инструкция по развертыванию Apache Hive 4.0 на Hadoop-кластере

---

## 1. Установка и настройка PostgreSQL

### 1.1. Установка PostgreSQL на NameNode
```bash
ssh team@ip_jn
ssh team-8-nn
sudo apt update
sudo apt install postgresql
```

### 1.2. Настройка базы данных
```bash
sudo -i -u postgres
psql

-- В интерактивной консоли PostgreSQL:
CREATE DATABASE metastore;
CREATE USER hive WITH PASSWORD 'hiveMegaPass';
GRANT ALL PRIVILEGES ON DATABASE metastore TO hive;
ALTER DATABASE metastore OWNER TO hive;
\q

exit
```

### 1.3. Конфигурация сетевого доступа
```bash
sudo vim /etc/postgresql/16/main/postgresql.conf
```
Добавить/изменить:
```ini
listen_addresses = 'team-8-nn'
```

```bash
sudo vim /etc/postgresql/16/main/pg_hba.conf
```
Добавить в конец файла:
```ini
host    metastore    hive    IP_NN/32    password
host    metastore    hive    IP_JN/32    password
```

Перезапустите PostgreSQL:
```bash
sudo systemctl restart postgresql
```

---

## 2. Установка Hive

### 2.1. Подготовка окружения
```bash
sudo apt install postgresql-client-16
sudo -i -u hadoop
wget https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
tar -xzvf apache-hive-4.0.0-alpha-2-bin.tar.gz
```

### 2.2. Установка JDBC-драйвера
```bash
cd apache-hive-4.0.0-alpha-2-bin/lib/
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

---

## 3. Конфигурация Hive

### 3.1. Настройка hive-site.xml
```bash
vim ../conf/hive-site.xml
```
Добавить конфигурацию:
```xml
<configuration>
    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://team-8-nn:5432/metastore</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hiveMegaPass</value>
    </property>
</configuration>
```

### 3.2. Настройка переменных окружения
```bash
vim ~/.profile
```
Добавить в конец файла:
```bash
export HIVE_HOME=/home/hadoop/apache-hive-4.0.0-alpha-2-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```

Применить изменения:
```bash
source ~/.profile
```

---

## 4. Подготовка HDFS
```bash
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
```

---

## 5. Инициализация схемы
```bash
cd apache-hive-4.0.0-alpha-2-bin/
bin/schematool -dbType postgres -initSchema
```

---

## 6. Запуск сервисов

### 6.1. Запуск HiveServer2
```bash
hive --hiveconf hive.server2.enable.doAs=false --hiveconf hive.security.authorization.enable service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```

---

## 7. Проверка работы

### 7.1. Подключение через Beeline
```bash
beeline -u jdbc:hive2://team-8-jn:5433 -n scott -p tiger
```
