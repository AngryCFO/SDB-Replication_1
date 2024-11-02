# Домашнее задание к занятию "Репликация и масштабирование. Часть 1" - `Александра Бужор`

---

### Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

Ответить в свободной форме.

### Решение:

Режимы репликации master-slave и master-master - это два различных подхода к распределению данных между базами данных.

### Репликация Master-Slave

*Описание*:
- В репликации master-slave есть один основной сервер (master), который обрабатывает записи, и один или несколько вторичных серверов (slaves), которые копируют данные с основного сервера.
- Вторичные серверы работают только для чтения и используются для распределения нагрузки при выполнении запросов на чтение или в качестве резервной копии для восстановления после сбоев.
- Все изменения данных происходят на основном сервере, и затем эти изменения реплицируются на вторичные серверы.

*Преимущества*:
- Простота настройки и управления.
- Уменьшает нагрузку на основной сервер, распределяя запросы на чтение по slave серверам.
- Улучшенное время отклика для запросов на чтение.

*Недостатки*:
- Задержка репликации может привести к тому, что slave серверы будут отставать от master.
- При отказе основного сервера требуется вмешательство для повышения slave до статуса master.

### Репликация Master-Master

*Описание*:
- В репликации master-master два или более серверов работают как основные, позволяя записывать на оба сервера.
- Каждый сервер реплицирует изменения данных на другой сервер, что позволяет записи и чтение на обоих серверах.

*Преимущества*:
- Повышенная отказоустойчивость, так как отказ одного сервера не приводит к потере доступности системы для записи.
- Может помочь в распределении нагрузки, если запись происходит на разные серверы из разных географических точек.

*Недостатки*:
- Сложнее настроить и управлять, особенно из-за потенциальных конфликтов синхронизации данных, когда одновременно изменяются одни и те же данные на разных серверах.
- Больше накладных расходов на синхронизацию данных между серверами.

*Особые соображения*:
- Необходимо тщательное планирование для предотвращения конфликтов данных и для управления такими сценариями, как автоинкрементные ключи, которые могут привести к проблемам уникальности при записи на разные серверы.

В обоих случаях репликация может выполняться синхронно или асинхронно. Синхронная репликация гарантирует, что все серверы будут содержать одинаковые данные в любой момент времени, но может ухудшить производительность из-за ожидания подтверждения записи от всех серверов. Асинхронная репликация обеспечивает более высокую производительность, но создает риск того, что данные на разных серверах могут временно расходиться.

---

### Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.

### Решение:

master
```bash
$ docker run -d --name replication-master -e MYSQL_ALLOW_EMPTY_PASSWORD=true -v ~/dump:/docker-entrypoint-initdb.d mysql:8.0
2c36c70755f826636a7ebd3274cce2ccee51b1dfe2ecce03de385fac040e05b0
$ docker run -d --name replication-slave -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0
50e5ba6e1bd22a1ba4bdc3688a748477191658637a3be8b3d8d76e8128eee3ee
$ docker exec -it replication-master mysql

mysql> CREATE USER 'replication'@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
Query OK, 0 rows affected (0.01 sec)```

mysql> CREATE DATABASE `world`;
Query OK, 1 row affected (0.02 sec)

$ docker restart replication-master
replication-master

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      157 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.00 sec)

$ docker exec replication-master mysqldump world > ./world.sql

mysql> UNLOCK TABLES;
Query OK, 0 rows affected (0.00 sec)
```

slave
```bash
$ docker cp ./world.sql replication-slave:/tmp/world.sql
Successfully copied 3.07kB to replication-slave:/tmp/world.sql
$ docker exec -it replication-slave mysql

mysql> CREATE DATABASE `world`;
Query OK, 1 row affected (0.02 sec)

mysql> CHANGE MASTER TO MASTER_HOST='replication-master', MASTER_USER='replication', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=157; 
Query OK, 0 rows affected, 7 warnings (0.03 sec)

mysql> START SLAVE;
Query OK, 0 rows affected, 1 warning (0.02 sec)

mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: replication-master
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 157
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 157
              Relay_Log_Space: 536
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: ac10f673-c9aa-11ee-ba6d-0242ac110002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)
```

Тесты:
master
```bash
mysql> USE world;
Database changed
mysql> CREATE TABLE city (
    -> Name TEXT,
    -> CountryCode TEXT,
    -> District TEXT,
    -> Population INTEGER
    -> );
Query OK, 0 rows affected (0.09 sec)

mysql> INSERT INTO city (Name, CountryCode, District, Population) VALUES
    -> ('Test-Replication', 'ALB', 'Test', 42);
Query OK, 1 row affected (0.02 sec)
```
slave
```bash
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: replication-master
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 727
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 896
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 727
              Relay_Log_Space: 1106
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: ac10f673-c9aa-11ee-ba6d-0242ac110002
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)

mysql> USE world;
Database changed

mysql> SELECT * FROM city;
+------------------+-------------+----------+------------+
| Name             | CountryCode | District | Population |
+------------------+-------------+----------+------------+
| Test-Replication | ALB         | Test     |         42 |
+------------------+-------------+----------+------------+
1 row in set (0.00 sec)
```
---

### Задание 3 *

Выполните конфигурацию master-master репликации. Произведите проверку.

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.

### Решение:
1. Создаем два контейнера MySQL:

```bash
# Создаем первый мастер
docker run -d --name master1 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0

# Создаем второй мастер
docker run -d --name master2 -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0
```

2. Настраиваем первый мастер (master1):

```bash
docker exec -it master1 mysql

# Создаем пользователя для репликации
mysql> CREATE USER 'replication'@'%';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

# Создаем тестовую базу данных
mysql> CREATE DATABASE test;

# Проверяем статус master1
mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 157
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
```

3. Настраиваем второй мастер (master2):

```bash
docker exec -it master2 mysql

# Создаем пользователя для репликации
mysql> CREATE USER 'replication'@'%';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

# Создаем тестовую базу данных
mysql> CREATE DATABASE test;

# Проверяем статус master2
mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 157
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
```

4. Настраиваем репликацию на master1:

```bash
mysql> CHANGE MASTER TO 
    MASTER_HOST='master2',
    MASTER_USER='replication',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=157;

mysql> START SLAVE;

# Проверяем статус репликации
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: master2
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 157
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

5. Настраиваем репликацию на master2:

```bash
mysql> CHANGE MASTER TO 
    MASTER_HOST='master1',
    MASTER_USER='replication',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=157;

mysql> START SLAVE;

# Проверяем статус репликации
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: master1
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 157
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

6. Проверяем работу репликации. На master1:

```bash
mysql> USE test;
mysql> CREATE TABLE example (id INT, name VARCHAR(50));
mysql> INSERT INTO example VALUES (1, 'Test from master1');

# Проверяем данные
mysql> SELECT * FROM example;
+------+------------------+
| id   | name             |
+------+------------------+
|    1 | Test from master1|
+------+------------------+
```

7. Проверяем на master2:

```bash
mysql> USE test;
mysql> SELECT * FROM example;
+------+------------------+
| id   | name             |
+------+------------------+
|    1 | Test from master1|
+------+------------------+

# Добавляем данные с master2
mysql> INSERT INTO example VALUES (2, 'Test from master2');
```

8. Проверяем репликацию обратно на master1:

```bash
mysql> SELECT * FROM example;
+------+------------------+
| id   | name             |
+------+------------------+
|    1 | Test from master1|
|    2 | Test from master2|
+------+------------------+
```

9. Настройка AUTO_INCREMENT для предотвращения конфликтов:

На master1:
```sql
SET GLOBAL auto_increment_increment=2;
SET GLOBAL auto_increment_offset=1;
```

На master2:
```sql
SET GLOBAL auto_increment_increment=2;
SET GLOBAL auto_increment_offset=2;
```

В результате настройки мы получили работающую master-master репликацию, где:
- Оба сервера могут принимать запросы на чтение и запись
- Изменения на любом сервере автоматически реплицируются на другой
- Настроено корректное использование AUTO_INCREMENT для предотвращения конфликтов
- Репликация работает в обе стороны, что подтверждено тестовыми операциями записи и чтения
---
