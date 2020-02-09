# Postgres cheatsheet, notices, etc. 

Найденные после прочтения книги заметки и примеры SQL запросов
PostgreSQL. Основы языка SQL: учеб. пособие / Е. П. Моргунов; под ред. Е. В. Рогова, П. В. Лузанова. — СПб.: БХВ-Петербург, 2018.

https://postgrespro.ru/education/books/sqlprimer
### Установка на Manjaro
https://wiki.archlinux.org/index.php/PostgreSQL_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)


- материализованные запросы кешируют результат запроса их нужно обновлять. Но как они реагируют на удаление таблиц из которых они взяли данные?
- SHOW search_path показывает к каким схемам постгрис обращается
- ограничение для поля не пустое ADD CHECK name <> ''
- USING для изменения поля с текстового на чиловое обозначение
- LIKE '___'  3 любых символа
- UNION два одинаковых запроса объединяются UNION ALL оставляет дубликаты у него есть еще 2 брата EXCEPT и INTERSECT
- Оконные функции для ранжирования отрабатывает после GROUP BY и HAVING
- count(some_field) считает не NULL значения
- Существуют так же триггеры и rules для учета изменений в таблице, но в пособии ничего о них нет
- NOW() возвращает начало старта транзакции, реальное время возвращает CLOCK_TIMESTAMP()
- В транзакции есть SAVEPOINT к которым можно откатиться и попробовать еще раз запустить команду в транзакции, количесво точек не ограничено, (знают случай когда их было 250.000)
```sql
SELECT seats.aircraft_code, model, seat_no, fare_conditions
FROM aircrafts ad
   LEFT JOIN seats on seats.aircraft_code = ad.aircraft_code
WHERE ad.model ~ '^Сессна'
```
SAME AS
```sql
SELECT seats.aircraft_code, model, seat_no, fare_conditions
FROM aircrafts ad, seats
WHERE seats.aircraft_code = ad.aircraft_code AND ad.model ~ '^Сессна'
```

на самом деле плохо работает и не эквивалент

```sql
SELECT a.aircraft_code as a_code,
       a.model,
       r.aircraft_code as r_code,
       count(r.aircraft_code) as num
FROM aircrafts a, routes r
where r.aircraft_code = a.aircraft_code
group by a_code, model, r_code
ORDER BY num DESC;
```

не тоже что вот это

```sql
SELECT a.aircraft_code as a_code,
       a.model,
       r.aircraft_code as r_code,
       count(r.aircraft_code) as num
FROM aircrafts a
LEFT JOIN routes r ON r.aircraft_code = a.aircraft_code
group by a_code, model, r_code
ORDER BY num DESC
```

но тоже что это

```sql
SELECT a.aircraft_code as a_code,
       a.model,
       r.aircraft_code as r_code,
       count(r.aircraft_code) as num
FROM aircrafts a
JOIN routes r ON r.aircraft_code = a.aircraft_code
group by a_code, model, r_code
ORDER BY num DESC
```


Оконная функция по дате.
```sql
SELECT
       b.book_ref,
       b.book_date,
       extract('month' from b.book_date) AS month,
       extract('day' from b.book_date) AS day,
       count(*) OVER (PARTITION BY date_trunc('month', b.book_date)  -- оконная функция
           ORDER BY b.book_date ASC) AS count                        
FROM ticket_flights tf
JOIN tickets t ON tf.ticket_no = t.ticket_no
JOIN bookings b on t.book_ref = b.book_ref
WHERE tf.flight_id = 2
ORDER BY b.book_date
```
Результат:
|book_ref| book_date              | month | day | count|
|--------|------------------------|-------|-----|------|
|A60039  | 2016-08-22 12:02:00+08 |8      | 22  |  1   |
|554340  | 2016-08-23 23:04:00+08 |8      | 23  |  2   |
|854C4C  | 2016-08-24 10:52:00+08 |8      | 24  |  5   |
|854C4C  | 2016-08-24 10:52:00+08 |8      | 24  |  5   |
|854C4C  | 2016-08-24 10:52:00+08 |8      | 24  |  5   |
|81D8AF  | 2016-08-25 10:22:00+08 |8      | 25  |  6   |

Обновление jsonb столбца
```sql
ALTER TABLE aircrafts ADD COLUMN specifications jsonb;

UPDATE aircrafts SET specifications = '{"crew": 2, "engines": {"type": "IAE V2500", "num": 2}}'::jsonb
WHERE aircraft_code = '320';
```
Выборка jsonb столбца
```sql
SELECT model, specifications->'engines'->'num' as engines from aircrafts_data;
```
эквивалент команде выше
```sql
SELECT model, specifications #> '{engines, num}' as engines from aircrafts_data;
```
Поиск с регулярными выражениями POSIX
```sql
select * from bookings.bookings WHERE book_ref !~ '^00';
```
```sql
SELECT seats.aircraft_code, model, seat_no, fare_conditions
FROM aircrafts ad, seats
WHERE seats.aircraft_code = ad.aircraft_code AND ad.model ~ '^Сессна';
```
Cколько маршрутов обслуживают самолеты каждого типа?
```sql
SELECT a.aircraft_code as a_code,
       a.model,
       r.aircraft_code as r_code,
       count(r.aircraft_code) as num
FROM aircrafts a
         LEFT JOIN routes r ON r.aircraft_code = a.aircraft_code
group by a_code, model, r_code
ORDER BY num DESC;
```
Oпределить число пассажиров, не пришедших на регистрацию билетов и, следовательно, не вылетевших в пункт назначения. Будем учитывать только рейсы, у которых фактическое время вылета не пустое, т.е. рейсы, имеющие статус Departed или Arrived.
```sql
SELECT count(*)
FROM (ticket_flights t JOIN flights f on t.flight_id = f.flight_id)
         LEFT JOIN boarding_passes b ON t.ticket_no = b.ticket_no AND t.flight_id = b.flight_id
WHERE f.actual_departure IS NOT NULL
  AND b.flight_id IS NULL;
```

Предположим, что для выработки финансовой стратегии нашей авиакомпании требуется распределение количества бронирований по диапазонам сумм с шагом в 100 тысяч рублей. Максимальная сумма в одном бронировании составляет 1 204 500 рублей. Учтем это при формировании диапазонов. 
Виртуальной таблице, создаваемой с помощью ключевого слова VALUES, присваивают имя с помощью ключевого слова AS. После имени в круглых скобках приводится список имен столбцов этой таблицы.
```sql
SELECT r.min, r.max, count(b.*)
FROM (VALUES (0, 100000), (100000, 200000), (200000, 1000000), (1000000, 10000000000), (1100000, 1100001)) AS r (min, max)
         left join bookings b
                   ON b.total_amount >= r.min AND b.total_amount < r.max
GROUP BY r.min, r.max
ORDER BY r.min DEsc;
```

```sql
SELECT *,
       rank() OVER (PARTITION BY rank() ORDER BY airport_code DESC)
FROM airports;
```
покажем, как можно сформировать диапазоны сумм бронирований с помощью рекурсивного общего табличного выражения:
```sql
WITH RECURSIVE ranges (min, max) AS
                   (VALUES (0, 100000)
                   UNION ALL
                       SELECT min + 100000, max + 100000
                       FROM ranges
                       WHERE max < (SELECT max(total_amount) FROM bookings)
                   )
SELECT * FROM ranges;
```
Теперь давайте скомбинируем рекурсивное общее табличное выражение с выборкой из таблицы bookings:
|min      | max     | count  |
|---------|---------|--------|
|0        | 100000  | 198314 |
|100000   | 200000  | 46943  |
|200000   | 300000  | 11916  |
|300000   | 400000  | 3260   |
|400000   | 500000  | 1357   |
|500000   | 600000  | 681    |
|600000   | 700000  | 222    |
|700000   | 800000  | 55     |
|800000   | 900000  | 24     |
|900000   | 1000000 | 11     |
|1000000  | 1100000 | 4      |
|1100000  | 1200000 | 0      |
|1200000  | 1300000 | 1      |
```sql
WITH RECURSIVE ranges (min, max) AS
                   (VALUES (0, 100000)
                    UNION ALL
                    SELECT min + 100000, max + 100000
                    FROM ranges
                    WHERE max < (SELECT max(total_amount) FROM bookings)
                   )
SELECT r.min, r.max, count(b.*)
FROM bookings b
RIGHT JOIN ranges r ON b.total_amount >= r.min AND b.total_amount < r.max
GROUP BY r.min, r.max
ORDER BY r.min;
```
Ответить на вопрос о том, каковы максимальные и минимальные цены билетов
на все направления, может такой запрос
```sql
SELECT f.departure_city, f.arrival_city, max(tf.amount), min(tf.amount)
FROM flights_v f
JOIN ticket_flights tf ON f.flight_id = tf.flight_id
GROUP BY departure_city, arrival_city;
```

## ИЗМЕНЕНИЕ ДАННЫХ
```sql
CREATE TEMP TABLE aircrafts_tmp AS SELECT * FROM aircrafts WITH NO DATA;
ALTER TABLE aircrafts_tmp ADD PRIMARY KEY (aircraft_code);
ALTER TABLE aircrafts_tmp ADD UNIQUE (model);
CREATE TEMP TABLE aircrafts_log AS SELECT * FROM aircrafts WITH NO DATA;
```
```sql
ALTER TABLE aircrafts_log ADD COLUMN when_add timestamp;
ALTER TABLE aircrafts_log ADD COLUMN operation text;
```
```sql
SELECT * FROM aircrafts_tmp;
SELECT * FROM aircrafts_log;
```
Общее табличное выражениею
Позволяет вставлять в две таблицы разом и причем разную инфу
WITH add_row как переменная, может быть переиспользована
```sql
WITH add_row AS (
    INSERT INTO aircrafts_tmp
        SELECT * FROM aircrafts
        RETURNING * -- возвращает все строки добавленные в таблицу
)
INSERT
INTO aircrafts_log
SELECT add_row.aircraft_code, add_row.model, add_row.range, current_timestamp, 'INSERT'
FROM add_row;
```
ON CONFLICT предусматривает два варианта действий.
1. Oтменять добавление новой строки, для которой имеет место конфликт значений ключевых атрибутов, и при этом не порождать сообщения об ошибке.
```sql
WITH add_row AS
         ( INSERT INTO aircrafts_tmp
             VALUES ( 'SU9', 'Sukhoi SuperJet-100', 3000 )
             ON CONFLICT DO NOTHING
             RETURNING *
         )
INSERT INTO aircrafts_log
SELECT add_row.aircraft_code, add_row.model, add_row.range,
       current_timestamp, 'INSERT'
FROM add_row;
```

Укажем конкретный столбец для проверки конфликтующих значений. Пусть это будет aircraft_code, т. е. первичный ключ. Для упрощения команды не будем использовать общее табличное выражение. Добавляемая строка будет конфликтовать с существующей строкой как по столбцу aircraft_code, так и по столбцу model.

```sql
INSERT INTO aircrafts_tmp
VALUES ('SU9', 'Sukhoi SuperJet-100', 3000) ON CONFLICT ( aircraft_code )
DO NOTHING
RETURNING *;
```

2. Замена операции добавления новой строки операцией обновления существующей строки, с которой конфликтует добавляемая строка.

Предложение ON CONFLICT DO UPDATE гарантирует атомарное выполнение операции вставки или обновления строк. Атомарность означает, что проверка наличия конфликта и последующее обновление выполняются как неделимая операция, т.е. другие транзакции не могут изменить значение столбца, вызывающее конфликт, так, чтобы в результате конфликт исчез и уже стало возможным выполнить операцию INSERT, а не UPDATE, или, наоборот, в случае отсутствия конфликта он вдруг появился, и уже операция INSERT стала бы невозможной.

Такая атомарная операция даже имеет название UPSERT — «UPDATE или INSERT».
```sql
INSERT INTO aircrafts_tmp
VALUES ( 'SU9', 'Sukhoi SuperJet', 3000 )
ON CONFLICT ON CONSTRAINT aircrafts_tmp_pkey -- первичный ключ, если вставляем строку с таким же ключом
    DO UPDATE SET model = excluded.model,    -- указываем какие поля обновить
                  range = excluded.range     -- специальная таблица excluded, которая поддерживается самой СУБД. В этой таблице хранятся все строки, предлагаемые для вставки в рамках текущей команды INSERT
RETURNING *;
```


Для массового ввода строк в таблицы используется команда COPY. Эта команда может копировать данные из файла в таблицу. Причем, в качестве файла может служить и стандартный ввод. Хотя в этом разделе пособия мы, в основном, говорим о вставке строк в таблицы, но нужно сказать и о том, что эта команда может также копировать данные из таблиц в файлы и на стандартный вывод. Можно даже импортировать CSV файл.

## Обновление строк в таблицах

Команда UPDATE предназначена для обновления данных в таблицах. Начнем с того, что покажем, как и при изучении команды INSERT, как можно организовать запись выполненных операций в журнальную таблицу.

Напомним, что выведенное сообщение относится непосредственно к внешнему запросу, в котором выполняется операция INSERT, добавляющая строку в журнальную таблицу.

Конечно, если бы строка в таблице aircrafts_tmp не была успешно обновлена, тогда предложение RETURNING * не возвратило бы внешнему запросу ни одной строки, и, следовательно, тогда просто не было бы данных для формирования новой строки в таблице aircrafts_log.

При использовании команды UPDATE в общем табличном выражении нужно учитывать, что главный запрос может получить доступ к обновленным данным только через временную таблицу, которую формирует предложение RETURNING:
```sql
WITH update_row AS
         (UPDATE aircrafts_tmp
             SET range = range * 1.2
             WHERE model ~ '^Bom'
             RETURNING *
         )
INSERT
INTO aircrafts_log
SELECT ur.aircraft_code,
       ur.model,
       ur.range,
       current_timestamp,
       'UPDATE'
FROM update_row ur;
```

Теперь представим команду, которая и будет добавлять новую запись о продаже билета и увеличивать в таблице tickets_directions значение счетчика проданных билетов.
Здесь интересная конструкция WHERE

```sql
WITH sell_ticket AS
         (INSERT INTO ticket_flights_tmp
             (ticket_no, flight_id, fare_conditions, amount)
             VALUES ('1234567890123', 30829, 'Economy', 12800)
             RETURNING *
         )
UPDATE tickets_directions td
SET last_ticket_time = current_timestamp,
    tickets_num      = tickets_num + 1
WHERE (td.departure_city, td.arrival_city) =
      (SELECT departure_city, arrival_city
       FROM flights_v
       WHERE flight_id = (SELECT flight_id FROM sell_ticket)
      );
```

Представим другой вариант этой команды. Его отличие от первого варианта в том, что для определения обновляемой строки в таблице tickets_directions используется операция соединения таблиц.

Здесь в главном запросе UPDATE присутствует предложение FROM, однако в этом предложении указывается только представление flights_v, а таблицу tickets_directions в предложение FROM включать не нужно, хотя она и участвует в выполнении соединения таблиц.

Конечно, в предложении SET присваивать новые значения можно толькоатрибутам таблицы tickets_directions,
поскольку именно она приведена в предложении UPDATE.

Пока не понял как это работает

```sql
WITH sell_ticket AS
         (INSERT INTO ticket_flights_tmp
             (ticket_no, flight_id, fare_conditions, amount)
             VALUES ('1234567890123', 7757, 'Economy', 3400)
             RETURNING *
         )
UPDATE tickets_directions td
SET last_ticket_time = current_timestamp,
    tickets_num      = tickets_num + 1
FROM flights_v f
WHERE td.departure_city = f.departure_city
  AND td.arrival_city = f.arrival_city
  AND f.flight_id = (SELECT flight_id FROM sell_ticket);
```

## УДАЛЕНИЕ

Покажем, как можно организовать запись выполненных операций в журнальную таблицу.

```sql
WITH delete_row AS
         (DELETE FROM aircrafts_tmp
             WHERE model ~ '^Bom'
             RETURNING *
         )
INSERT
INTO aircrafts_log
SELECT dr.aircraft_code,
       dr.model,
       dr.range,
       current_timestamp,
       'DELETE'
FROM delete_row dr;
```

Предположим, что руководство авиакомпании решило удалить из парка самолетов машины компаний Boeing и Airbus, имеющие наименьшую дальность полета.

В общем табличном выражении с помощью условия model ~'^Airbus' OR model ~'^Boeing' в предложении WHERE отберем модели только компаний Boeing и Airbus. Затем воспользуемся оконной функцией rank и произведем ранжирование моделей каждой компании по возрастанию дальности полета. Те модели, ранг которых окажется равным 1, будут иметь наименьшую дальность полета. В предложении USING сформируем соединение таблицы aircrafts_tmp с временной таблицей min_ranges, а затем в предложении WHERE зададим условия для отбора строк.

Тут самолеты ранжируются по дальности полета отдельно у каждого производителя

```sql
WITH min_ranges AS
         (SELECT aircraft_code, left(model, 6),
                 rank() OVER (
                     PARTITION BY left(model, 6)
                     ORDER BY range
                     ) AS rank
          FROM aircrafts
          WHERE model ~ '^Аэробус'
             OR model ~ '^Боинг'
         )
SELECT * FROM min_ranges;
```

# Транзакции
### 1. Read Uncommitted. 

Это самый низкий уровень изоляции. Согласно стандарту SQL на этом уровне допускается чтение «грязных» (незафиксированных) данных.
Однако в Postgres требования, предъявляемые к этому уровню, более строгие, чем в стандарте: чтение «грязных» данных на этом уровне не допускается.

Изменили одну строку и не закомиттили в одном терминале, второй терминал может видеть это изменение Postgres Read Uncommitted === Read Committed, так что можно сказать Read Uncommitted не поддерживается

### 2. Read Committed. 
Не допускается чтение «грязных» (незафиксированных) данных.
Таким образом, в PostgreSQL уровень Read Uncommitted совпадает с уровнем Read Committed.
Транзакция может видеть только те незафиксированные изменения данных,
которые произведены в ходе выполнения ее самой.

Изменили одну строку и не закомиттили в одном терминале, второй терминал НЕ может видеть это изменение

В первом терминале изменили строку (UPDATE), но не закомиттили, второй терминал при запуске команды UPDATE не завершает команду пока в первом терминале не будет выполнен COMMIT или ROLLBACK

Здесь есть non-repeatable read
В первом терминале выполняем в транзацкии ```SELECT * FROM table```, получем 10 строк, не коммитим
Во втором терминале ```DELETE FROM table WHERE id <= 3```, удалили 3 строки, COMMIT
В первом снова выполняем ```SELECT * FROM table``` получем 7 строк. non-repeatable read во всей красе

### 3. Repeatable Read. 

Не допускается чтение «грязных» (незафиксированных) данных и неповторяющееся чтение. 

В Postgres на этом уровне не допускается также фантомное чтение.
Таким образом, реализация этого уровня является более строгой, чем того требует стандарт SQL. Это не противоречит стандарту.

Неповторяющееся чтение (non-repeatable read). При повторном чтении тех же самых данных в рамках одной транзакции оказывается, что другая транзакция успела изменить и зафиксировать эти данные. В результате тот же самый запрос выдает другой результат.
Фантомное чтение (phantom read). Транзакция повторно выбирает множество строк в соответствии с одним и тем же критерием. В интервале времени между выполнением этих выборок другая транзакция добавляет новые строки и успешно фиксирует изменения. В результате при выполнении повторной выборки в первой транзакции может быть получено другое множество строк.

Приложения, использующие этот уровень изоляции, должны быть готовы к тому, что придется выполнять транзакции повторно. Это объясняется тем, что транзакция, использующая этот уровень изоляции, создает снимок данных не перед выполнением каждого запроса, а только однократно, перед выполнением первого запроса транзакции.

Поэтому транзакции с этим уровнем изоляции не могут изменять строки, которые были изменены другими завершившимися транзакциями уже после создания снимка. Вследствие этого Postgres не позволит зафиксировать транзакцию, которая попытается изменить уже измененную строку.

Повторный запуск может потребоваться только для транзакций, которые вносят изменения в данные. Для транзакций, которые только читают данные, повторный запуск никогда не требуется.

Здесь нет non-repeatable read

В первом терминале выполняем в транзацкии SELECT * FROM table, получем 10 строк, не коммитим

Во втором терминале ```DELETE FROM table WHERE id <= 3```, удалили 3 строки, COMMIT
В первом снова выполняем SELECT * FROM table получем 10 строк
На первом терминале ничего не изменилось: фантомные строки не видны, и также не видны изменения в уже существующих строках. Это объясняется тем, что снимок данных выполняется на момент начала выполнения первого запроса транзакции.

Но здесь есть ошибки сериализации:
На первом терминале UPDATE table SET range = range + 100 WHERE id = 1, не коммитим
На втором терминале UPDATE table SET range = range + 200 WHERE id = 1, не коммитим
Команда UPDATE на втором терминале ожидает завершения первой транзакции.
Перейдя на первый терминал, завершим первую транзакцию: COMMIT
Перейдя на второй терминал, увидим сообщение об ошибке:
ОШИБКА: не удалось сериализовать доступ из-за параллельного изменения

Поскольку обновление, произведенное в первой транзакции, не было зафиксировано на момент начала выполнения первого (и, в данном частном случае, единственного) запроса во второй транзакции, то возникает эта ошибка. Это объясняется вот чем.
При выполнении обновления строки команда UPDATE во второй транзакции видит, что строка уже изменена. На уровне изоляции Repeatable Read снимок данных создается на момент начала выполнения первого запроса транзакции и в течение транзакции уже не меняется, т.е. новая версия строки не считывается, как это делалось на уровне Read Committed. Но если выполнить обновление во второй транзакции без повторного считывания строки из таблицы, тогда будет иметь место потерянное обновление, что недопустимо. В результате генерируется ошибка, и вторая транзакция откатывается.
Мы вводим команду COMMIT на втором терминале, но Postgres выполняет не фиксацию (COMMIT), а откат: ROLLBACK
Если выполним запрос SELECT *, то увидим, что было проведено только изменение в первой транзакции

### 4. Serializable. 
Не допускается ни один из феноменов, перечисленных выше, в том числе и аномалии сериализации.

Самый высший уровень изоляции транзакций — Serializable. Транзакции могут работать параллельно точно так же, как если бы они выполнялись последовательно одна за другой.
Однако, как и при использовании уровня Repeatable Read, приложение должно быть готово к тому, что придется перезапускать транзакцию, которая была прервана системой из-за обнаружения зависимостей чтения/записи между транзакциями. Группа транзакций может быть параллельно выполнена и успешно зафиксирована в том случае, когда результат их параллельного выполнения был бы эквивалентен результату выполнения этих транзакций при выборе одного из возможных вариантов их упорядочения, если бы они выполнялись последовательно, одна за другой.

Для проведения эксперимента создадим специальную таблицу, в которой будет всего два столбца: один — числовой, а второй — текстовый. 
Назовем эту таблицу modes.

|num | mode|
|----|-----|
|1   | LOW |
|2   | HIGH|

На первом терминале начнем транзакцию и запустим UPDATE modes SET mode = 'HIGH' WHERE mode = 'LOW'
На втором терминале тоже начнем транзакцию UPDATE modes SET mode = 'LOW' WHERE mode = 'HIGH'
Изменение, произведенное в первой транзакции, вторая транзакция не видит, потому что на уровне изоляции Serializable каждая транзакция работает с тем снимком базы данных, который был сделан непосредственно перед выполнением ее первого оператора.
Поэтому обновляется только одна строка, та, в которой значение поля mode было равно HIGH изначально.
Обратите внимание, что обе команды UPDATE были выполнены, ни одна из них не ожидает завершения другой транзакции.
Посмотрим, что получилось в первой транзакции:
| num| mode|
|----|-----|
|  2 | HIGH|
|  1 | HIGH|

Во второй транзакции:
| num | mode |
|-----|------|
|  1  | LOW  |
|  2  | LOW  |

Сначала завершим транзакцию на первом терминале: COMMIT
А потом на втором терминале: COMMIT
ОШИБКА: не удалось сериализовать доступ из-за зависимостей чтения/записи между транзакциями
ПОДРОБНОСТИ: Reason code: Canceled on identification as a pivot, during commit attempt.
ПОДСКАЗКА: Транзакция может завершиться успешно при следующей попытке.

Какое же изменение будет зафиксировано? То, которое сделала транзакция, первой выполнившая фиксацию изменений.
```sql
SELECT * FROM modes;
```
|num | mode |
|----|------|
|  2 | HIGH |
|  1 | HIGH |

Таким образом, параллельное выполнение двух транзакций сериализовать не удалось. Почему?
Если обратиться к определению концепции сериализации, то нужно рассуждать так.
Если бы была зафиксирована и вторая транзакция, тогда в таблице modes содержались бы такие строки:

| num | mode |
|-----|------|
|  1  | HIGH |
|  2  | LOW  |

Но этот результат не соответствует результату выполнения транзакций ни при одном из двух возможных вариантов их упорядочения, если бы они выполнялись последовательно.
Следовательно, с точки зрения концепции сериализации эти транзакции невозможно сериализовать.
Покажем это, выполнив транзакции последовательно.

Предварительно необходимо пересоздать таблицу modes и вернуть ее исходное состояние.
Теперь обе транзакции можно выполнять на одном терминале. Первый вариант их упорядочения такой:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE modes SET mode = 'HIGH' WHERE mode = 'LOW';
COMMIT;

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE modes SET mode = 'LOW' WHERE mode = 'HIGH';
COMMIT;

Проверим, что получилось:
```sql
SELECT * FROM modes;
```
| num | mode |
|-----|------|
|  2  | LOW  |
|  1  | LOW  |

Во втором варианте упорядочения поменяем транзакции местами. Предварительно вернув таблицу в исходное состояние.
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE modes SET mode = 'LOW' WHERE mode = 'HIGH';
COMMIT

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE modes SET mode = 'HIGH' WHERE mode = 'LOW' RETURNING *;
COMMIT
```sql
SELECT * FROM modes;
```

Теперь результат отличается от того, который был получен при реализации первого варианта упорядочения транзакций.
| num | mode |
|-----|------|
|  1  | HIGH |
|  2  | HIGH |

Изменение порядка выполнения транзакций приводит к разным результатам. Однако, если бы при параллельном выполнении транзакций была зафиксирована и втораяиз них, то полученный результат не соответствовал бы ни одному из продемонстрированных возможных результатов последовательного выполнения транзакций. Такимобразом, выполнить сериализацию этих транзакций невозможно. Обратите внимание, что вторая команда UPDATE в обоих случаях обновляет не одну строку, а две.

Блокировки строк

Первый терминалб выполнить ```SELECT * FROM table FOR UPDATE``` в транзакции Read Committed
Если выполнить на втором терминале ```SELECT * FROM table FOR UPDATE```, команда не выполнится пока в первом терминале транзакция не закоммитится

Блокировка таблицы
На первом терминале
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
LOCK TABLE table_name IN ACCESS EXCLUSIVE MODE;
```
На втором терминале выполните совершенно «безобидную» команду:
```sql
SELECT * FROM table_name WHERE model ~ '^Air';
```
Вы увидите, что выполнение команды SELECT на втором терминале будет задержано.
Прервите транзакцию на первом терминале командой ROLLBACK.
Вы увидите, что на втором терминале команда будет успешно выполнена.
