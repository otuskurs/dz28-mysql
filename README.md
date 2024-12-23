MySQL: Backup + Репликация 

Цель задания:

В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp

Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
```
| bookmaker |
| competition |
market |
| odds |
| outcome
```
Настроить GTID репликацию.
варианты которые принимаются к сдаче
- рабочий вагрантафайл
- скрины или логи SHOW TABLES


Входе выполнения ДЗ был написан ansible-playbook mysql.yml который настраивает и подготавливает mysql на серверха master и slave.

Для настройки баз необходимо сначала на мастере подготавить таблицу , развернуть дамп и подготовить ее к реплике 

```
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}' - посмотреть стоковый пароль 

mysql> ALTER USER USER() IDENTIFIED BY 'YourStrongPassword'; - смена пароля 

mysql> SELECT @@server_id;

mysql> SHOW VARIABLES LIKE 'gtid_mode';

mysql> CREATE DATABASE bet;

mysql -uroot -p -D bet < /vagrant/bet.dmp

mysql> USE bet;

mysql> SHOW TABLES;

CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';

SELECT user,host FROM mysql.user where user='repl';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
```

Отдельно хочу вынести комманду которая делает дапм базы. Только не рекомендую заливать на слейф потому , что при запуске он выдает ошибку что таблица есть и не может читать с мастера. 

```
mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```

На слейве в конфигах надо поменять id - /etc/my.cnf.d/01-basics.cnf  server-id = 2

А в /etc/my.cnf.d/05-binlog.cnf добавляем строки.

```
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event
```
Настраиваем slave
```
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.91.95", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
```
И проверяем репликацию 

- на мастере делаем 
```
mysql> USE bet;
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(2,'fonbet');
mysql> SELECT * FROM bookmaker;
```

![Скорость iperf3 в режиме tap](https://github.com/otuskurs/dz28-mysql/blob/main/%D1%81%D0%BA%D1%80%D0%B8%D0%BD%D1%8B/master-2.png)

И проверяем slave

```
SELECT * FROM bookmaker;
```
![Скорость iperf3 в режиме tun](https://github.com/Dogmatic41/otus/blob/main/43.mysql/images/slave.png)
