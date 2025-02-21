Вот полная инструкция в одном `.md` файле:

```markdown
# Инструкция по развертыванию HDFS на 4 виртуальных машинах

## Обозначения узлов
| Роль             | Имя узла       | IP-адрес (пример) |
|-------------------|----------------|-------------------|
| Jump Node        | `team-8-jn`    | `ip_jn`           |
| NameNode         | `team-8-nn`    | `ip_nn`           |
| DataNode 1       | `team-8-dn-00` | `ip_dn-00`        |
| DataNode 2       | `team-8-dn-01` | `ip_dn-01`        |

---

## 1. Настройка SSH и hosts-файлов

### На Jump Node (`ip_jn`)
```bash
ssh team@ip_jn
ssh-keygen -t ed25519
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# Копирование ключей на все узлы
scp ~/.ssh/authorized_keys ip_nn:/home/team/.ssh/
scp ~/.ssh/authorized_keys ip_dn-00:/home/team/.ssh/
scp ~/.ssh/authorized_keys ip_dn-01:/home/team/.ssh/

# Редактирование /etc/hosts
sudo vim /etc/hosts
```
Добавить:
```
127.0.0.1 team-8-jn
ip_nn team-8-nn
ip_dn-00 team-8-dn-00
ip_dn-01 team-8-dn-01
```

### На NameNode (`ip_nn`)
```bash
ssh ip_nn
sudo adduser hadoop
sudo vim /etc/hosts
```
Добавить:
```
ip_nn team-8-nn
ip_jn team-8-jn
ip_dn-00 team-8-dn-00
ip_dn-01 team-8-dn-01
```

### На DataNode 1 (`ip_dn-00`)
```bash
ssh ip_dn-00
sudo adduser hadoop
sudo vim /etc/hosts
```
Добавить:
```
127.0.0.1 team-8-dn-00
ip_jn team-8-jn
ip_nn team-8-nn
ip_dn-01 team-8-dn-01
```

### На DataNode 2 (`ip_dn-01`)
```bash
ssh ip_dn-01
sudo adduser hadoop
sudo vim /etc/hosts
```
Добавить:
```
127.0.0.1 team-8-dn-01
ip_jn team-8-jn
ip_nn team-8-nn
ip_dn-00 team-8-dn-00
```

---

## 2. Установка Hadoop

### На всех узлах
```bash
sudo -i -u hadoop
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
tar -xzvf hadoop-3.4.0.tar.gz
```

### Настройка окружения
В файл `~/.profile` добавить:
```bash
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
Применить изменения:
```bash
source ~/.profile
```

---

## 3. Конфигурация Hadoop

### Основные файлы конфигурации
1. `hadoop-env.sh`:
   ```bash
   export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
   ```

2. `core-site.xml`:
   ```xml
   <configuration>
     <property>
       <name>fs.defaultFS</name>
       <value>hdfs://team-8-nn:9000</value>
     </property>
   </configuration>
   ```

3. `hdfs-site.xml`:
   ```xml
   <configuration>
     <property>
       <name>hdfs.replication</name>
       <value>3</value>
     </property>
   </configuration>
   ```

4. `workers`:
   ```
   team-8-nn
   team-8-dn-00
   team-8-dn-01
   ```

### Распространение конфигурации
```bash
# На Jump Node от имени hadoop
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-8-nn:$HADOOP_HOME/etc/hadoop/
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-8-dn-00:$HADOOP_HOME/etc/hadoop/
scp hadoop-env.sh core-site.xml hdfs-site.xml workers team-8-dn-01:$HADOOP_HOME/etc/hadoop/
```

---

## 4. Запуск HDFS

### На NameNode
```bash
ssh team-8-nn
sudo -i -u hadoop
cd $HADOOP_HOME

# Форматирование HDFS
bin/hdfs namenode -format

# Запуск служб
sbin/start-dfs.sh
```

### Проверка работы
```bash
jps # Должны отображаться процессы: NameNode, DataNode, SecondaryNameNode
hdfs dfsadmin -report
```


---
**Примечание**: Все команды выполняются от имени пользователя `hadoop`, если не указано иное. Замените `ip_jn`, `ip_nn`, `ip_dn-00`, `ip_dn-01` на реальные IP-адреса ваших ВМ.
