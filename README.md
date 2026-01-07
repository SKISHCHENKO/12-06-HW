# Домашнее задание к занятию «Репликация и масштабирование. Часть 1» - Кищенко Сергей FOPS-41


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

# Домашнее задание к занятию «Репликация и масштабирование. Часть 1»

Выполнено с использованием DBeaver

## Задание 1. 

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.  

Ответить в свободной форме.  
  
## Решение 1

  a) Репликация master–slave  

Это наиболее распространённая схема, в которой есть один главный сервер и один или несколько подчинённых.  

Master — главный сервер, только он принимает записи  
Slave — ведомый сервер, только читает данные, получая изменения от мастера  
Master записывает все изменения в binlog (binary log)  
Slave считывает binlog и сохраняет у себя в relay log  

В этой конфигурации:  
- Master (ведущий) — основной сервер. Все операции записи (INSERT, UPDATE, DELETE) происходят только на нём.  
- Slave (ведомый) — копия мастера. Он получает все изменения от мастера и обслуживает запросы на чтение (SELECT).  
Принцип работы:  
1. Master записывает все изменения в специальный журнал — binary log.  
2. Slave копирует этот журнал к себе в relay log.  
3. Slave применяет изменения из своего журнала, обновляя собственные данные.  
   
Эта схема эффективна для систем, где количество операций чтения значительно превышает количество операций записи.  
Типы репликации:  
- покомандная — репликация SQL-запросов  
- построчная — репликация изменённых данных (строк)  
  

   b) Репликация master–master

  В этой конфигурации все серверы равноправны — каждый является одновременно и мастером, и слейвом для другого. Запросы на запись или
чтение можно отправлять на любой из серверов, и изменения будут распространены на все остальные.  
  Такая схема повышает отказоустойчивость: если один из мастеров выйдет из строя, система продолжит работу, используя оставшиеся серверы.    

Два (или больше) сервера:   
Каждый одновременно master и slave   
Можно писать и читать на любом   

Master-1 реплицируется в Master-2  
Master-2 реплицируется в Master-1  

Фактически это две master-slave репликации в разные стороны  
  

## Задание 2. 

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.  

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.  

## Решение 2

Master: VM1 rmq01, IP 192.168.56.104, контейнер mysql8 (MySQL 8.0)  
Slave: VM2 rmq02, IP 192.168.56.105, контейнер mysql8_slave (MySQL 8.0)  
Сеть между ВМ: host-only 192.168.56.0/24 (машины пингуются)   

Обе ВМ:  
Ubuntu Jammy  
MySQL 8.0 в Docker  
соединены по сети 192.168.56.0/24  

На master:  

docker exec -it mysql8 sh -lc 'cat > /etc/mysql/conf.d/replication.cnf <<EOF  
[mysqld]  
server-id=1  
log-bin=mysql-bin  
binlog-format=ROW  
EOF'  
docker restart mysql8  
 
docker exec -it mysql8 mysql -uroot -pRootPass123 -e "  
SHOW VARIABLES LIKE 'server_id';  
SHOW VARIABLES LIKE 'log_bin';  
SHOW VARIABLES LIKE 'binlog_format';  

docker exec -it mysql8 mysql -uroot -pRootPass123 -e "  
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'ReplPass123';  
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';  
FLUSH PRIVILEGES;  
"  

docker exec -it mysql8 mysql -uroot -pRootPass123 -e "SHOW MASTER STATUS\G"  

![Задание 2](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task2_1.png)

На slave  

docker run -d \  
  --name mysql8_slave \  
  -e MYSQL_ROOT_PASSWORD=RootPass123 \  
  -p 3306:3306 \  
  -v mysql8_slave_data:/var/lib/mysql \  
  mysql:8.0  

docker exec -it mysql8_slave sh -lc 'cat > /etc/mysql/conf.d/replication.cnf <<EOF  
[mysqld]  
server-id=2  
read-only=1  
EOF'  
docker restart mysql8_slave  

docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "  
SHOW VARIABLES LIKE 'server_id';  
SHOW VARIABLES LIKE 'read_only';  
"  
docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "  
CHANGE REPLICATION SOURCE TO  
SOURCE_HOST='192.168.56.104',  
SOURCE_USER='repl',  
SOURCE_PASSWORD='ReplPass123',  
SOURCE_LOG_FILE='mysql-bin.000001',  
SOURCE_LOG_POS=871;  
START REPLICA;  
"  

sergk@rmq02:~$ docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "  
SHOW VARIABLES LIKE 'server_id';  
SHOW VARIABLES LIKE 'read_only';  
"
![Задание 2](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task2_2.png)

docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "SHOW REPLICA STATUS\G"

![Задание 2](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task2_3.png)

На master создали БД/таблицу и вставили строку  
docker exec -it mysql8 mysql -uroot -pRootPass123 -e "  
CREATE DATABASE IF NOT EXISTS test_db;  
USE test_db;  
CREATE TABLE IF NOT EXISTS test_table (id INT PRIMARY KEY, name VARCHAR(50));  
INSERT INTO test_table VALUES (1,'Master Record')  
ON DUPLICATE KEY UPDATE name=VALUES(name);  
SELECT * FROM test_table;  
"

![Задание 2](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task2_4.png)

На slave убедились, что данные появились  
docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "  
SELECT * FROM test_db.test_table;  
"

![Задание 2](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task2_5.png)

## Задание 3.  

Выполните конфигурацию master-master репликации. Произведите проверку.  

Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов. 

## Решение 3  

Master: VM1 rmq01, IP 192.168.56.104, контейнер mysql8 (MySQL 8.0)   
Slave: VM2 rmq02, IP 192.168.56.105, контейнер mysql8_slave (MySQL 8.0)  
Сеть между ВМ: host-only 192.168.56.0/24 (машины пингуются)  

Обе ВМ:  
Ubuntu Jammy  
MySQL 8.0 в Docker  
соединены по сети 192.168.56.0/24  

Поднимаем контейнеры  

На rmq01  
docker run -d \  
  --name mysql8 \  
  -e MYSQL_ROOT_PASSWORD=RootPass123 \  
  -p 3306:3306 \  
  -v mysql8_data:/var/lib/mysql \  
  mysql:8.0  

docker exec -it mysql8 sh -lc 'cat > /etc/mysql/conf.d/replication.cnf <<EOF  
[mysqld]   
server-id=1  
log-bin=mysql-bin  
binlog-format=ROW  
auto_increment_increment=2  
auto_increment_offset=1   
replicate-ignore-db=mysql  
EOF'  

docker restart mysql8  

docker exec -it mysql8 mysql -uroot -pRootPass123 -e "  
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'ReplPass123';  
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';  
FLUSH PRIVILEGES;  
"  

![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_1.png)

На rmq02

docker run -d \  
  --name mysql8_slave \  
  -e MYSQL_ROOT_PASSWORD=RootPass123 \  
  -p 3306:3306 \  
  -v mysql8_slave_data:/var/lib/mysql \  
  mysql:8.0  


docker exec -it mysql8_slave sh -lc 'cat > /etc/mysql/conf.d/replication.cnf <<EOF 
[mysqld]  
server-id=2  
log-bin=mysql-bin  
binlog-format=ROW  
auto_increment_increment=2  
auto_increment_offset=2  
replicate-ignore-db=mysql  
EOF'  

docker restart mysql8_slave  

docker exec -it mysql8_slave mysql -uroot -pRootPass123 -e "  
CREATE USER IF NOT EXISTS 'repl'@'%' IDENTIFIED BY 'ReplPass123';  
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';  
FLUSH PRIVILEGES;  
"  

![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_2.png)

Мастер статус на обоих:  

![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_3.png)
![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_4.png)

Параметры и подтверджение работы реплики:

![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_5.png)
![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_6.png)

Создание таблицы на первой машине и проверка видимости таблицы на второй машине:

![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_7.png)
![Задание 3](https://github.com/SKISHCHENKO/12-06-HW/blob/main/img/task3_8.png)