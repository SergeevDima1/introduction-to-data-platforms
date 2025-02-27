# Инструкция по развертыванию YARN на кластере Hadoop

---

## 1. Настройка Nginx прокси

### На Jump Node:
```bash
ssh team@ip_jn
cd /etc/nginx/sites-available

# Создание конфигов для проксирования
sudo cp default nn
sudo cp default ya
sudo cp default dh
```

### Конфигурационные файлы:

1. **nn** (для HDFS):
```nginx
server {
    listen 9870;
    server_name _;
    
    location / {
        proxy_pass http://team-8-nn:9870;
    }
}
```

2. **ya** (для YARN ResourceManager):
```nginx
server {
    listen 8088;
    server_name _;
    
    location / {
        proxy_pass http://team-8-nn:8088;
    }
}
```

3. **dh** (для History Server):
```nginx
server {
    listen 19888;
    server_name _;
    
    location / {
        proxy_pass http://team-8-nn:19888;
    }
}
```

### Активация конфигов:
```bash
sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/

sudo nginx -t
sudo systemctl reload nginx
```

---

## 2. Конфигурация YARN

### На всех узлах:
```bash
sudo -i -u hadoop
cd hadoop-3.4.0/etc/hadoop/
```

1. **yarn-site.xml**:
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>team-8-nn</value>
    </property>
</configuration>
```

2. **mapred-site.xml**:
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

### Распространение конфигурации:
```bash
# С Jump Node
scp yarn-site.xml mapred-site.xml team-8-nn:$HADOOP_HOME/etc/hadoop/
scp yarn-site.xml mapred-site.xml team-8-dn-00:$HADOOP_HOME/etc/hadoop/
scp yarn-site.xml mapred-site.xml team-8-dn-01:$HADOOP_HOME/etc/hadoop/
```

---

## 3. Запуск служб

### На NameNode:
```bash
ssh team-8-nn
sudo -i -u hadoop

# Запуск YARN
$HADOOP_HOME/sbin/start-yarn.sh

# Запуск History Server
mapred --daemon start historyserver
```

---

## 4. Проверка работоспособности

1. Локальный доступ через туннель:
```bash
ssh -L 9870:127.0.0.1:9870 \
    -L 8088:127.0.0.1:8088 \
    -L 19888:127.0.0.1:19888 \
    team@ip_jn
```
