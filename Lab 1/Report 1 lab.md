# Отчёт по лабораторной работе №1: базовая настройка PostgreSQL на Debian 
Велиев С.С. Группа ИС-22 

**Цель:** настроить окружение, установить PostgreSQL в Debian, освоить базовые приёмы администрирования системы и СУБД.  
**ОС:** Debian 12 (VM)  

---

## 1. Подготовка среды

Обновление индексов пакетов:

```bash
sudo apt update
sudo apt upgrade
````

> `apt update` обновляет *списки пакетов* (репозитории).

> `apt upgrade` обновляет *установленные пакеты* до новых версий.

---

## 2. Установка PostgreSQL

Установка PostgreSQL и доп. пакета:

```bash
sudo apt install -y postgresql postgresql-contrib
```

Проверка версии:

```bash
psql --version
```

---

## 3. Создание служебной учётной записи `postgres`

Проверка, что пользователь `postgres` создан в системе:

```bash
getent passwd postgres
```

Переход под пользователя `postgres`:

```bash
sudo -i -u postgres
```

Вход в psql:

```bash
psql
```

**Назначение `postgres`:**

* системный пользователь Linux, под которым работает сервис PostgreSQL;
* владеет каталогом данных PostgreSQL;
* имеет админские полномочия в СУБД (роль `postgres` обычно суперпользователь).

---

## 4. Поиск и изучение конфигурационных файлов

Получение путей к основным файлам:

```bash
sudo -i -u postgres psql -c "SHOW config_file;"
sudo -i -u postgres psql -c "SHOW hba_file;"
sudo -i -u postgres psql -c "SHOW data_directory;"
```

Результат:

* `config_file`: `/etc/postgresql/15/main/postgresql.conf`
* `hba_file`: `/etc/postgresql/15/main/pg_hba.conf`
* `data_directory`: `/var/lib/postgresql/15/main`

---

## 5. Изменения конфигурации PostgreSQL (подробно)

### 5.1 Редактирование `postgresql.conf`

Файл:

* `/etc/postgresql/15/main/postgresql.conf`

Открытие:

```bash
sudo nano /etc/postgresql/15/main/postgresql.conf
```

Внесённые изменения:

1. **Смена порта PostgreSQL на 5445**
   (т.к. подключение выполнялось с `-p 5445`)

```conf
port = 5445
```

2. **Разрешение слушать все сетевые интерфейсы** (для подключений из локальной сети):

```conf
listen_addresses = '*'
```

3. **Настройка логирования (logging)**:

```conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'all'
log_min_duration_statement = 0
```

Пояснение:

* `logging_collector=on` — включает запись логов в файлы;
* `log_directory='log'` — каталог логов (обычно внутри `data_directory`);
* `log_filename=...` — формат имени файлов;
* `log_statement='all'` — логируются все SQL-запросы;
* `log_min_duration_statement=0` — логируются все запросы независимо от времени выполнения.

### 5.2 Редактирование `pg_hba.conf`

Файл:

* `/etc/postgresql/15/main/pg_hba.conf`

Открытие:

```bash
sudo nano /etc/postgresql/15/main/pg_hba.conf
```

Внесённые изменения (разрешение сетевого доступа из локальной сети):

```conf
host    velbd     Vel     192.168.1.0/24     scram-sha-256
```

Пояснение формата строки:

* `host` — TCP/IP доступ по сети;
* `velbd` — имя базы;
* `Vel` — имя пользователя/роли;
* `192.168.1.0/24` — разрешённая подсеть;
* `scram-sha-256` — парольная аутентификация (SCRAM).

### 5.3 Применение изменений

После изменения конфигов выполнен перезапуск:

```bash
sudo systemctl restart postgresql
```

---

## 6. Управление сервисом (systemd)

Проверка статуса:

```bash
sudo systemctl status postgresql --no-pager
```

Включение автозапуска:

```bash
sudo systemctl enable postgresql
systemctl is-enabled postgresql
```

---

## 7. Создание тестового пользователя и базы данных

Переход под postgres:

```bash
sudo -i -u postgres
```

Создание роли `"Vel"`:

```bash
createuser --interactive
```

Ответы:

* role: `Vel`
* superuser: `n`
* create databases: `n`
* create roles: `n`

Задание пароля (в psql):

```sql
ALTER ROLE "Vel" WITH LOGIN PASSWORD '123';
```

Создание базы `velbd` с владельцем Vel:

```bash
createdb -O Vel velbd
```

Проверка подключения:

```bash
psql -U Vel -d velbd -h 127.0.0.1 -p 5445 -W
```

---

## 8. Схемы: public и test_schema

**Схема** — “папка/пространство имён” внутри базы данных.

**База данных** — контейнер, внутри которого находятся схемы и объекты.

Подключение под postgres к `velbd`:

```bash
sudo -i -u postgres psql -d velbd
```

Создание схемы:

```sql
CREATE SCHEMA test_schema;
```

Выдача прав пользователю `"Vel"`:

```sql
GRANT USAGE ON SCHEMA test_schema TO "Vel";
GRANT CREATE ON SCHEMA test_schema TO "Vel";
```

Проверка прав:

```sql
\dn+ test_schema
```


## 9. Работа с search_path

Проверка значения по умолчанию:

```sql
SHOW search_path;
```

Вывод:

```
"$user", public
```

Изменение search_path в текущей сессии:

```sql
SET search_path TO test_schema, public;
SHOW search_path;
```

Вывод:

```
test_schema, public
```

---

## 10. psql: базовые операции (public и test_schema)

Проверка текущего пользователя и базы (пример):

```sql
SELECT current_user, current_database();
```

### 10.1 Таблица в `public`

Создание таблицы:

```sql
CREATE TABLE public.clients (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  phone TEXT UNIQUE
);
```

Вставка данных:

```sql
INSERT INTO public.clients(name, phone) VALUES
('Levan', '3917641'),
('Phillip', '66917641');
```

Выборка:

```sql
SELECT * FROM public.clients;
```

Обновление:

```sql
UPDATE public.clients SET phone='333' WHERE name='Phillip';
```

### 10.2 Таблица в `test_schema`

Создание таблицы:

```sql
CREATE TABLE test_schema.orders (
  id SERIAL PRIMARY KEY,
  client_name TEXT NOT NULL,
  total NUMERIC(10,2) NOT NULL
);
```

Вставка:

```sql
INSERT INTO test_schema.orders(client_name, total) VALUES
('Petr', 10.50),
('Petr', 99.99);
```

Выборка:

```sql
SELECT * FROM test_schema.orders;
```

Выход:

```sql
\q
```

---

## 11. Скрипт (таблица в конкретной схеме)

Создание файла:

```bash
nano schema_demo.sql
```

Содержимое:

```sql
CREATE TABLE IF NOT EXISTS test_schema.clothes (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL
);

INSERT INTO test_schema.clothes(title) VALUES ('T-Shirt'), ('Jeans');

SELECT * FROM test_schema.clothes;
```

Запуск:

```bash
psql -h 127.0.0.1 -p 5445 -U "Vel" -d velbd -f schema_demo.sql
```

Результат:

```
CREATE TABLE
INSERT 0 2
 id |  title
----+---------
  1 | T-Shirt
  2 | Jeans
(2 rows)
```

---

## 12. Сетевой доступ и firewall (UFW)

Установка UFW:

```bash
sudo apt update
sudo apt install -y ufw
```

Разрешение SSH, включение UFW и проверка статуса:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
sudo ufw status
```

Открытие порта PostgreSQL (5445):

```bash
sudo ufw allow 5445/tcp
```

---

## 13. Журналирование (logging) — проверка

После изменения настроек логов выполнен перезапуск:

```bash
sudo systemctl restart postgresql
```

Просмотр логов сервиса:

```bash
sudo journalctl -u postgresql --no-pager -n 50
```

---

## 14. Назначение ролей и прав + наследование

Подключение под postgres к базе `velbd`:

```bash
sudo -i -u postgres psql -d velbd
```

Создание ограниченной роли:

```sql
CREATE ROLE limited_user LOGIN PASSWORD 'limit';
```

Выдача минимальных прав:

```sql
GRANT CONNECT ON DATABASE velbd TO limited_user;
GRANT USAGE ON SCHEMA public TO limited_user;
GRANT SELECT ON public.clients TO limited_user;
```

Подключение под limited_user для теста:

```sql
\c "host=127.0.0.1 port=5445 dbname=velbd user=limited_user password=limit"
```

Тест:

```sql
SELECT * FROM public.clients; -- работает
INSERT INTO public.clients(name) VALUES ('Test'); -- отказывает
```

Выход:

```sql
\q
```

### 14.1 Наследование прав через роль readonly

Подключение под postgres:

```bash
sudo -i -u postgres psql
```

Создание групповой роли:

```sql
CREATE ROLE readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
```

Назначение роли readonly пользователю limited_user:

```sql
GRANT readonly TO limited_user;
ALTER ROLE limited_user INHERIT;
```

Выход:

```sql
\q
```

---

## Итог

В ходе лабораторной работы:

* установлен PostgreSQL 15 на Debian;
* найдены и изучены конфигурационные файлы `postgresql.conf`, `pg_hba.conf`;
* внесены изменения: `port=5445`, `listen_addresses='*'`, включено логирование в файлы;
* в `pg_hba.conf` добавлено правило доступа из локальной сети `192.168.1.0/24` для базы `velbd` и пользователя `Vel` с `scram-sha-256`;
* создан пользователь `"Vel"` и база `velbd`, выполнены подключения и базовые SQL-операции;
* создана схема `test_schema`, выданы права `USAGE/CREATE`, продемонстрирована работа со схемами и `search_path`;
* выполнен скрипт `schema_demo.sql`, создающий таблицу в конкретной схеме;
* настроен UFW (разрешены SSH и порт PostgreSQL);
* создан `limited_user` с ограниченными правами, показано наследование прав через роль `readonly`.