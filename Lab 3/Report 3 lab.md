# Отчёт по лабораторной работе №3: расширенные возможности и оптимизация PostgreSQL на Debian  
Велиев С.С. Группа ИС-22  

**Цель:** получить опыт использования продвинутых функций PostgreSQL: настройка параметров производительности, индексы и планы запросов, функции и триггеры, VACUUM/ANALYZE и статистика.  
**ОС:** Debian 12 (VM)  
**СУБД:** PostgreSQL (порт `5445`)  
**База:** `velbd`  


## 1. Оптимизация конфигурации PostgreSQL

### 1.1 Определение объёма оперативной памяти VM

Команда:

```bash
free -h
```

Результат:

```
admin1@debian-post:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           9.7Gi       1.5Gi       7.2Gi        46Mi       1.3Gi       8.2Gi
Swap:          974Mi          0B       974Mi
```

Параметры `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size` зависят от RAM. При некорректных значениях либо не будет прироста производительности, либо возможна избыточная нагрузка на память.

---

### 1.2 Переход под пользователя postgres

```bash
sudo -i -u postgres
```

Системный пользователь Linux `postgres` используется для администрирования и обслуживания PostgreSQL.

---

### 1.3 Поиск файла конфигурации postgresql.conf

```bash
psql -p 5445 -d velbd -c "SHOW config_file;"
```

Команда показывает точный путь к активному конфигу `postgresql.conf`.

---

### 1.4 Изменение параметров в postgresql.conf

Открытие файла:

```bash
nano /etc/postgresql/*/main/postgresql.conf
```

Установленные значения:

```conf
shared_buffers = 2GB
effective_cache_size = 7GB
work_mem = 32MB
maintenance_work_mem = 1GB
```

#### Назначение параметров и обоснование

**`shared_buffers = 2GB`**  
- **Что это:** память под буферный кеш PostgreSQL (страницы таблиц/индексов).  
- **Зачем:** снижает обращения к диску при повторных чтениях.  
- **Почему 2GB:** при RAM ≈ 9.7GiB разумно выделить около 20–25% под `shared_buffers`, оставляя место ОС для page cache.

**`effective_cache_size = 7GB`**  
- **Что это:** оценка доступного кеша (ОС + `shared_buffers`) для оптимизатора.  
- **Зачем:** влияет на выбор плана (чаще индексные планы, если данные ожидаются в кеше).  
- **Почему 7GB:** примерно 70% RAM — типичный ориентир для Linux.

**`work_mem = 32MB`**  
- **Что это:** память на одну операцию sort/hash внутри запроса на одну сессию.  
- **Зачем:** уменьшает использование временных файлов на диске для сортировок/хешей.  
- **Почему 32MB:** баланс между ускорением и безопасностью (важно помнить, что `work_mem` суммируется между операциями и сессиями).

**`maintenance_work_mem = 1GB`**  
- **Что это:** память на операции обслуживания (VACUUM, CREATE INDEX и др.).  
- **Зачем:** ускоряет вакуум и построение индексов.  
- **Почему 1GB:** операции обслуживания выполняются не постоянно, поэтому можно выделить больше.

---

### 1.5 Применение изменений и перезапуск сервиса

```bash
exit
sudo systemctl restart postgresql
sudo systemctl status postgresql --no-pager
```

Фрагмент вывода:

```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) ...
Feb 14 11:26:26 debian-post systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Feb 14 11:26:26 debian-post systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
```

На Debian служба `postgresql.service` может иметь статус `active (exited)`, потому что является “обёрткой” над конкретными кластерами PostgreSQL.

---

### 1.6 Проверка параметров через SHOW

```bash
sudo -i -u postgres
psql -p 5445 -d velbd -c "SHOW shared_buffers;"
psql -p 5445 -d velbd -c "SHOW work_mem;"
psql -p 5445 -d velbd -c "SHOW maintenance_work_mem;"
psql -p 5445 -d velbd -c "SHOW effective_cache_size;"
```

Результат:

```
shared_buffers = 2GB
work_mem = 32MB
maintenance_work_mem = 1GB
effective_cache_size = 7GB
```

**Вывод:** настройки применились.

---  

## 2. Индексы + EXPLAIN / EXPLAIN ANALYZE

### 2.1 Почему не использовалась таблица public.clients
Таблица `public.clients` уже имеет индексы:
- `clients_pkey` (PRIMARY KEY по `id`);
- `clients_phone_key` (UNIQUE по `phone`).

Если тестировать запросы по `id` или `phone`, индекс используется сразу, и сравнение “до/после” будет неинформативным. Поэтому создана отдельная таблица `sales`, чтобы показать изменение плана запроса после создания индекса.

---

### 2.2 Создание таблицы sales

```sql
CREATE TABLE IF NOT EXISTS public.sales (
  id bigserial PRIMARY KEY,
  customer_id int NOT NULL,
  product_id int NOT NULL,
  price numeric(10,2) NOT NULL,
  created_at timestamp NOT NULL DEFAULT now()
);
```

Индексируемым полем выбран `customer_id`, так как по нему выполняется фильтрация в `WHERE`.

---

### 2.3 Обновление статистики (ANALYZE)

```sql
ANALYZE public.sales;
```

Оптимизатор использует статистику для оценки селективности и выбора плана (Seq Scan vs Index Scan).

---

### 2.4 EXPLAIN ANALYZE до индекса

```sql
EXPLAIN ANALYZE
SELECT * FROM public.sales
WHERE customer_id = 4242;
```

Фрагмент результата:

```
Parallel Seq Scan on sales
Filter: (customer_id = 4242)
Rows Removed by Filter: 166666
Execution Time: 21.844 ms
```

#### Разбор плана
- **Seq Scan**: последовательное чтение всей таблицы с фильтрацией строк по условию.  
- **Parallel Seq Scan**: PostgreSQL подключает параллельных workers, чтобы ускорить чтение при отсутствии индекса.  
- **Rows Removed by Filter**: сколько строк было прочитано, но отброшено условием — показатель “лишней работы” при Seq Scan.

---

### 2.5 Создание индекса

```sql
CREATE INDEX idx_sales_customer_id ON public.sales(customer_id);
```

**Что делает индекс:** создаёт структуру (обычно B-tree), где значения `customer_id` упорядочены, и для каждого значения есть ссылки на строки таблицы. Это позволяет находить нужные строки без полного просмотра таблицы.

---

### 2.6 EXPLAIN ANALYZE после индекса

```sql
EXPLAIN ANALYZE
SELECT * FROM public.sales
WHERE customer_id = 4242;
```

Фрагмент результата:

```
Bitmap Index Scan on idx_sales_customer_id
Bitmap Heap Scan on sales
Execution Time: 0.163 ms
```

#### Почему стало быстрее и что означает Bitmap Scan
- **Bitmap Index Scan**: по индексу собирается “карта” подходящих строк.  
- **Bitmap Heap Scan**: затем PostgreSQL читает только нужные блоки таблицы.  
- Это часто эффективнее, чем “прыгать” по таблице для каждой строки отдельно, когда совпадений несколько.

**Сравнение:** время выполнения уменьшилось примерно с ~21.8 ms до ~0.16 ms, план сменился с `Parallel Seq Scan` на использование индекса.

---  

## 3. Хранимые функции (PL/pgSQL)

### 3.1 Таблица payments

```sql
CREATE TABLE IF NOT EXISTS public.payments (
  id bigserial PRIMARY KEY,
  amount numeric(10,2) NOT NULL,
  created_at timestamp NOT NULL DEFAULT now()
);
```

---

### 3.2 Функция add_payment (проверка значения + вставка)

```sql
CREATE OR REPLACE FUNCTION public.add_payment(p_amount numeric)
RETURNS text
LANGUAGE plpgsql
AS $$
BEGIN
  IF p_amount < 0 THEN
    RETURN 'Ошибка: отрицательное значение';
  END IF;

  INSERT INTO public.payments(amount) VALUES (p_amount);
  RETURN 'Запись добавлена';
END;
$$;
```

Данная функция реализует простое бизнес-правило: отрицательные суммы не добавляются, вместо этого возвращается текст ошибки.

---

### 3.3 Проверка работы

```sql
SELECT public.add_payment(100.50);
SELECT public.add_payment(-10);
SELECT * FROM public.payments ORDER BY id DESC LIMIT 5;
```

**Результат:**
- `100.50` → “Запись добавлена”, строка появляется в `payments`;
- `-10` → “Ошибка: отрицательное значение”, вставки нет.

---  

## 4. Триггеры (проверка бизнес-правил)

### 4.1 Таблица products

```sql
CREATE TABLE IF NOT EXISTS public.products (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  price numeric(10,2) NOT NULL
);
```

### 4.2 Функция-триггер

```sql
CREATE OR REPLACE FUNCTION public.trg_check_price()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  IF NEW.price < 0 THEN
    RAISE EXCEPTION 'Цена не может быть отрицательной: %', NEW.price;
  END IF;
  RETURN NEW;
END;
$$;
```

Триггер проверяет поле `NEW.price` (новое значение). При нарушении правила — прекращает операцию ошибкой.

### 4.3 Создание триггера BEFORE INSERT/UPDATE

```sql
DROP TRIGGER IF EXISTS check_price_before_ins_upd ON public.products;

CREATE TRIGGER check_price_before_ins_upd
BEFORE INSERT OR UPDATE ON public.products
FOR EACH ROW
EXECUTE FUNCTION public.trg_check_price();
```

### 4.4 Тестирование

```sql
INSERT INTO public.products(name, price) VALUES ('Phone', 500);
INSERT INTO public.products(name, price) VALUES ('BadItem', -1); -- ошибка
UPDATE public.products SET price = -10 WHERE name='Phone';       -- ошибка
```

**Результат:**
```sql
INSERT 0 1
ERROR:  Цена не может быть отрицательной: -1.00
CONTEXT:  PL/pgSQL function trg_check_price() line 4 at RAISE
ERROR:  Цена не может быть отрицательной: -10.00
CONTEXT:  PL/pgSQL function trg_check_price() line 4 at RAISE
```

---  

## 5. VACUUM, ANALYZE и статистика

### 5.1 Проверка autovacuum и параметров

```sql
SHOW autovacuum;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_scale_factor;
```

Результат:

- `autovacuum = on`
- `autovacuum_naptime = 1min`
- `autovacuum_vacuum_scale_factor = 0.2`
- `autovacuum_analyze_scale_factor = 0.1`

Autovacuum автоматически удаляет “мёртвые” версии строк (MVCC) и запускает анализ статистики по мере накопления изменений.

---

### 5.2 Ручной VACUUM (ANALYZE) для sales

```sql
VACUUM (ANALYZE) public.sales;
```

**Пояснение:**
- **VACUUM** освобождает место от “мёртвых” строк и предотвращает разрастание таблиц;  
- **ANALYZE** обновляет статистику для оптимизатора.

---

### 5.3 Статистика по таблицам (pg_stat_user_tables)

```sql
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  vacuum_count,
  autovacuum_count,
  analyze_count,
  autoanalyze_count,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**Что показывают поля:**
- `n_live_tup` / `n_dead_tup` — оценка “живых” и “мёртвых” строк;
- `vacuum_count` — сколько раз делали VACUUM вручную;
- `autovacuum_count` — сколько раз vacuum делался автоматически;
- `last_*` — даты/время последних операций.

По результатам видно, что для таблицы `sales` выполнялись операции vacuum/analyze (manual и auto), что подтверждает работу VACUUM и autovacuum.

---

### 5.4 Статистика по индексам (pg_stat_all_indexes)

```sql
SELECT
  schemaname,
  relname,
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_all_indexes
WHERE schemaname IN ('public', 'test_schema')
ORDER BY idx_scan DESC;
```

**Пояснение:**
- `idx_scan` — сколько раз индекс использовался в планах;
- `idx_tup_read` — сколько записей индекс прочитал;
- `idx_tup_fetch` — сколько записей реально извлекли из таблицы по ссылкам индекса.

Индекс `idx_sales_customer_id` имеет `idx_scan > 0`, значит реально участвует в запросах.

---  

## Итог

В ходе лабораторной работы №3:

- настроены параметры производительности PostgreSQL с учётом RAM VM (~9.7GiB), проверены через `SHOW`;  
- показано влияние индекса на план запроса: до индекса `Parallel Seq Scan`, после — `Bitmap Index Scan/Bitmap Heap Scan`, время выполнения существенно уменьшилось;  
- создана функция `add_payment()` на PL/pgSQL для проверки входных данных и условной вставки;  
- реализован триггер, запрещающий отрицательные цены с `RAISE EXCEPTION`;  
- изучены autovacuum-параметры, выполнен `VACUUM (ANALYZE)` и просмотрена статистика по таблицам и индексам через `pg_stat_user_tables` и `pg_stat_all_indexes`.

