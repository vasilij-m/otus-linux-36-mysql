## Задание

Развернуть базу из дампа и настроить репликацию
Цель: В результате выполнения ДЗ студент развернет базу из дампа и настроит репликацию.
В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
| bookmaker |
| competition |
| market |
| odds |
| outcome

**\*** Настроить GTID репликацию

варианты которые принимаются к сдаче
- рабочий вагрантафайл
- скрины или логи SHOW TABLES
**\*** конфиги
**\*** пример в логе изменения строки и появления строки на реплике 

## Выполнение задания

Установим репозиторий Percona и сам пакет Percona-Server-57:

```
[root@master ~]# yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
```
```
[root@master ~]# yum install -y Percona-Server-server-57
```

Копируем конфиги в /etc/my.cnf.d/ и запускаем службу mysqld:

```
[root@master ~]# cp /vagrant/conf.d/* /etc/my.cnf.d/
[root@master ~]# systemctl start mysqld
[root@master ~]# systemctl enable mysqld
```

Найдем автоматически сгенерированный пароль для пользователя root в mysql:

```
[root@master ~]# grep 'root@localhost:' /var/log/mysqld.log | awk '{print $11}'
%o?Dyka)v5_i
```

Подключимся к mysql с этим паролем и поменяем его:

```
[root@master ~]# mysql -uroot -p'%o?Dyka)v5_i'
mysql> alter user user() identified by 'MySqlP@ssw0rd';
Query OK, 0 rows affected (0.00 sec)
```

Для подключения к mysql без указания учетных данных пользователя root создадим файл `/root/.my.cnf` со следующим содержимым и установим на него права `0600`:

```
[root@master ~]# cat .my.cnf 
[client]
user=root
password='MySqlP@ssw0rd'
```

Для настройки GTID репликации в файле настроек `my.cnf` нужно добавить параметр `gtid-mode = On`, а также задать различающиеся id'шники на master (`server-id = 1`) и slave (`server-id = 2`).

После установки параметра `server-id` нужно рестартануть службу `mysqld`. Проверить значение параметра можно следующим образом:

```
mysql> select @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```

Также проверим, что GTID режим репликации включен:

```
[root@master ~]# mysql -e "show variables like 'gtid_mode';"
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
```

Создадим тестовую базу bet и загрузим в неё имеющийся дамп:

```
[root@master ~]# mysql -e "create database bet;"
[root@master ~]# mysql -D bet < /vagrant/bet.dmp
```

Проверим, что импорт дампа выполнен успешно:

```
[root@master ~]# mysql -e "use bet; show tables;"
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
```

Создадим пользователя для репликации с правами на репликацию:

```
[root@master ~]# mysql -e "create user 'repl'@'%' identified by 'P@ssw0rdForRepl';"
[root@master ~]# mysql -e "grant replication slave on *.* to 'repl'@'%' identified by 'P@ssw0rdForRepl';"
```

Проверим, что пользователя имеет право на репликацию:

```
[root@master ~]# mysql -e "select user,host,repl_slave_priv from mysql.user where user='repl';"
+------+------+-----------------+
| user | host | repl_slave_priv |
+------+------+-----------------+
| repl | %    | Y               |
+------+------+-----------------+
```

По заданию необходимо настроить репликацию таким образом, чтобы в неё входили следующие таблицы: bookmaker, competition, market, odds, outcome. Для этого сделаем дамп базы без таблиц events_on_demand и v_same_event для последующей его загрузки  на slave:

```
[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event > master.sql
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events. 
[root@master ~]# ll
total 972
-rw-------. 1 root root   5570 Apr 30 22:09 anaconda-ks.cfg
-rw-r--r--. 1 root root 977685 Oct 17 20:11 master.sql
-rw-------. 1 root root   5300 Apr 30 22:09 original-ks.cfg
```

Зальем дамп с мастера на слейв с помощью утилиты Netcat, предварительно установив её на обоих серверах:

```
[root@master ~]# yum install -y nc

# Открываем порт на слейве для Netcat с перенаправлением в файл /tmp/master.sql
[root@slave ~]# nc -l 5217 > /tmp/master.sql

# Заливаем дамп с мастера на слейв:
[root@master ~]# nc 192.168.11.20 5217 < /root/master.sql
```

Настраивем слейв для репликации.

Приведем файл `/etc/my.cnf.d/05-binlog.cnf` к следующему виду и рестартанем сервис mysqld:

```
[root@slave ~]# cat /etc/my.cnf.d/05-binlog.cnf
[mysqld]
log-bin = mysql-bin
expire-logs-days = 7
max-binlog-size = 16M
binlog-format = "MIXED"

# GTID replication config
log-slave-updates = On
gtid-mode = On
enforce-gtid-consistency = On

replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event
```

Перед импортом дампа на слейве выполняем команду `reset master`, иначе mysql выдаст ошибку, что глобальная переменная `GTID_EXECUTED` должна быть пустой перед установкой переменной `GTID_PURGED`:

```
[root@slave ~]# mysql -e "reset master;"
```

Импортируем дамп с мастера и убеждаемся, что база bet не содержит лишних таблиц:

```
[root@slave ~]# mysql -e "source /tmp/master.sql;"
[root@slave ~]# mysql -e "show databases like 'bet';"
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
[root@slave ~]# mysql -e "use bet; show tables;"
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
```

Теперь подключаем и запускаем слейв:

```
[root@slave ~]# mysql -e "change master to master_host='192.168.11.10', master_port=3306, master_user='repl', master_password='P@ssw0rdForRepl', master_auto_position=1;"
[root@slave ~]# mysql -e "start slave;"
[root@slave ~]# mysql -e "show slave status\G;"
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.11.10
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 119302
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 414
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 119302
              Relay_Log_Space: 621
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
                  Master_UUID: 9dfab19f-106a-11eb-909a-5254004d77d3
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 9dfab19f-106a-11eb-909a-5254004d77d3:1-38
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version:
```

Следующие строки этого вывода говорят о том, что репликация работает в режиме GTID и игнорируются таблицы по заданию:

```
Slave_IO_State: Waiting for master to send event
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
Retrieved_Gtid_Set: 
Executed_Gtid_Set: 9dfab19f-106a-11eb-909a-5254004d77d3:1-38
```

Проверим репликацию, изменив таблицу bookmaker  на мастере:

```
[root@master ~]# mysql -e "use bet; select * from bookmaker;"
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
[root@master ~]# mysql -e "use bet; insert into bookmaker (id,bookmaker_name) values(1,'1xbet');"
[root@master ~]# mysql -e "use bet; select * from bookmaker;"
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

Проверим на слейве:

```
[root@slave ~]# mysql -e "use bet; select * from bookmaker;"
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

## Проверка задания

**Requirements:**
* **Перед проверкой задания необходимо установить плагин vagrant-vbguest: `vagrant plugin install vagrant-vbguest`.**
* **Ansible 2.7**

1. Выполнить `vagrant up`.

2. Подключиться к серверу master `vagrant ssh master`:
  * Вывести содержимое таблицы bookmaker:
  ```
  mysql -e "use bet; select * from bookmaker;"
  ```
  * Добавить запись 1xbet в таблицу bookmaker:
  ```
  mysql -e "use bet; insert into bookmaker (id,bookmaker_name) values(1,'1xbet');"
  ```
3. Подключиться к серверу slave `vagrant ssh slave`:
  * Вывести содержимое таблицы bookmaker, убедившись, что новая запись в ней присутствует:
  ```
  mysql -e "use bet; select * from bookmaker;"
  ```












