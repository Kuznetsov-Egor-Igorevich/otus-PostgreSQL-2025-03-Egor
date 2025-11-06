# Урок "Блокировки"

Включаю журналирование ожидания в блокировках и журналирование ошибок блокировок:

```pgsql
ALTER SYSTEM SET log_lock_failures = 'on';
ALTER SYSTEM SET log_lock_waits = 'on';
ALTER SYSTEM SET deadlock_timeout = 200;
```

Перезагружаю кластер:
```bash
sudo systemctl restart postgresql
```

Создаю тестовую таблицу с тестовыми данными:
```pgsql
CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
```

Эмулирую ситуацию блокировки для записи в журнал, в первой сессии делаю:
```pgsql
BEGIN;

SELECT pg_backend_pid(); --359663

UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
```


Во второй сессии делаю:
```pgsql
BEGIN;

SELECT pg_backend_pid() --359595

CREATE INDEX ON accounts(acc_no);
```

Проверяю лог:
```bash
tail -n 100 /var/log/postgresql/postgresql-18-main.log
```

В ответ получаю:
```bash
2025-11-06 17:39:12.102 MSK [359595] postgres@otus СООБЩЕНИЕ:  процесс 359595 продолжает ожидать в режиме ShareLock блокировку "отношение 16634 базы данных 16469" в течение 200.289 мс
2025-11-06 17:39:12.102 MSK [359595] postgres@otus ПОДРОБНОСТИ:  Process holding the lock: 359663. Wait queue: 359595.
2025-11-06 17:39:12.102 MSK [359595] postgres@otus ОПЕРАТОР:  CREATE INDEX ON accounts(acc_no);
```

Моделирую ситуацию обновления одной и той же строки в 3 разных сеансах:
```pgsql
BEGIN;

SELECT pg_backend_pid(); --359595

UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
```

сеанс2:
```pgsql
BEGIN;

SELECT pg_backend_pid(); --359663

UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
```

сеанс3:
```pgsql
BEGIN;  

SELECT pg_backend_pid(); --360098

UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
```

Проверяю pg_locks:
```pgsql
SELECT locktype, relation::REGCLASS, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass order by pid;
```

В ответ получаю:
```pgsql
relation	accounts	RowExclusiveLock	true	359595	{}
relation	accounts	RowExclusiveLock	true	359663	{359595}
tuple	accounts	ExclusiveLock	true	359663	{359595}
relation	accounts	RowExclusiveLock	true	360098	{359663}
tuple	accounts	ExclusiveLock	false	360098	{359663}
```

Тут видно что каждая новая транзакция блокируется предыдущей.


Моделирую взаимные блокировки:
```pgsql
BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
```

Сессия2:
```pgsql
BEGIN;
UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
```

Сессия3:
```pgsql
BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
```

Сессия1:
```pgsql
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
```

Смотрю лог:
```bash
tail -n 100 /var/log/postgresql/postgresql-18-main.log
```

В ответ получаю:
```bash
2025-11-06 18:12:56.055 MSK [359595] postgres@otus ОШИБКА:  обнаружена взаимоблокировка
2025-11-06 18:12:56.055 MSK [359595] postgres@otus ПОДРОБНОСТИ:  Процесс 359595 ожидает в режиме ExclusiveLock блокировку "кортеж (0,2) отношения 16634 базы данных 16469"; заблокирован процессом 360098.
	Процесс 360098 ожидает в режиме ShareLock блокировку "транзакция 1200375"; заблокирован процессом 359663.
	Процесс 359663 ожидает в режиме ShareLock блокировку "транзакция 1200373"; заблокирован процессом 359595.
	Процесс 359595: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
	Процесс 360098: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
	Процесс 359663: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
```

Из журнала видно что произошла взаимоблокировка.

Отвечаю на последний вопрос, ИИ говорит следующее: "Да, две такие транзакции могут заблокировать друг друга, если их выполнение приведет к конфликту блокировок на уровне страниц или строк, даже при отсутствии условия WHERE. Взаимоблокировка (deadlock) возникает из-за того, что транзакции захватывают блокировки в разном порядке."
