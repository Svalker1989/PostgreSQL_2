### Задача 1
Используя Docker, поднимите инстанс PostgreSQL (версию 13). Данные БД сохраните в volume.  
`docker-compose -f docker-compose.yml up -d`  
[docker-compose.yml]()  
Подключитесь к БД PostgreSQL, используя psql.  
Воспользуйтесь командой \? для вывода подсказки по имеющимся в psql управляющим командам.  
Найдите и приведите управляющие команды для:  
  
* вывода списка БД,  
`\l`  
* подключения к БД,  
`SELECT * FROM pg_stat_activity;`  
* вывода списка таблиц,  
`\dt`  
* вывода описания содержимого таблиц,  
`\d имя таблицы`  
* выхода из psql.  
`\q`  
### Задача 2
Используя psql, создайте БД test_database.  
Изучите бэкап БД.  
Восстановите бэкап БД в test_database.  
` psql -U postgres -d test_database -f /tmp/postgres/backup/test_dump.sql postgres`  
Перейдите в управляющую консоль psql внутри контейнера.  
Подключитесь к восстановленной БД и проведите операцию ANALYZE для сбора статистики по таблице.  
`analyze verbose orders;`  
Используя таблицу pg_stats, найдите столбец таблицы orders с наибольшим средним значением размера элементов в байтах.  
Приведите в ответе команду, которую вы использовали для вычисления, и полученный результат.  
`select attname from pg_stats where (tablename = 'orders') and (avg_width = (select max (avg_width) from pg_stats where tablename = 'orders'));`  
![]()  

### Задача 3
Архитектор и администратор БД выяснили, что ваша таблица orders разрослась до невиданных размеров и поиск по ней занимает долгое время. Вам как успешному выпускнику курсов DevOps в Нетологии предложили провести разбиение таблицы на 2: шардировать на orders_1 - price>499 и orders_2 - price<=499.  
Предложите SQL-транзакцию для проведения этой операции.  
Создаем партиции:  
```SQL
CREATE TABLE orders_1 (  
	check (price>499) 
	)INHERITS (orders);
CREATE INDEX orders_1_price ON orders_1 (price);
CREATE TABLE orders_2 (  
	check (price<=499) 
	)INHERITS (orders);	
CREATE INDEX orders_2_price ON orders_2 (price);
```
Распределяем записи из родительской таблицы по партициям:  
```SQL
WITH x AS (  
    DELETE FROM ONLY orders      
        WHERE price > 499 RETURNING *)
INSERT INTO orders_1  
    SELECT * FROM x;
	
WITH x AS (  
    DELETE FROM ONLY orders      
        WHERE price <= 499 RETURNING *)
INSERT INTO orders_2  
    SELECT * FROM x;
```
Можно ли было изначально исключить ручное разбиение при проектировании таблицы orders?  
Чтобы избежать ручного разбиения таблицы orders необходимо будет создать функцию автоматического партицирования с заданными условиями, и поместить выполнение данной функции в триггер "BEFORE INSERT". Но в данном случае все равно предварительно нужно будет создать партиции с условиями.  
  
Задача 4
Используя утилиту pg_dump, создайте бекап БД test_database.
`pg_dump test_database > test_database.sql`
  
Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца title для таблиц test_database?  
  
Для того чтобы в бэкап попала таблица с  уникальными значениями столбца title, нужно создать новую таблицу:  
```SQL
create table title_uniq INHERITS (orders);
```
Далее делаем выборку уникальных значений по столцу title и записываем её в новую таблицу:  
```SQL
WITH x AS (  
    select distinct on (title) id,title,price from orders RETURNING *)
INSERT INTO title_uniq 
    SELECT * FROM x;
```
Создаем бэкап только этой таблицы (ключ -t *имя таблицы*), либо если нужен бэкап все базы, то исключаем из него исходную таблицу с дублирующимися значениями (ключ	-T *имя таблицы*)  
`pg_dump	-t title_uniq test_database > test_database.sql`  
