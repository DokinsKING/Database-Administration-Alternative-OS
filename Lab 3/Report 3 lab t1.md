free -h

-------------------
admin1@debian-post:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           9.7Gi       1.5Gi       7.2Gi        46Mi       1.3Gi       8.2Gi
Swap:          974Mi          0B       974Mi
-------------------

sudo -i -u postgres

psql -p 5445 -d velbd -c "SHOW config_file;"

nano /etc/postgresql/*/main/postgresql.conf


------------------------------------------------------
shared_buffers = 2GB                    # min 128kB

effective_cache_size = 7GB

work_mem = 32MB                         # min 64kB

maintenance_work_mem = 1GB              # min 1MB
-----------------------------------------------------

exit
sudo systemctl restart postgresql
sudo systemctl status postgresql --no-pager


--------------------------------------------
admin1@debian-post:~$ sudo systemctl restart postgresql
admin1@debian-post:~$ sudo systemctl status postgresql --no-pager
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Sat 2026-02-14 11:26:26 EST; 3s ago
    Process: 9536 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 9536 (code=exited, status=0/SUCCESS)
        CPU: 2ms

Feb 14 11:26:26 debian-post systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
Feb 14 11:26:26 debian-post systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.
-------------------------------------------------------------------------------------------


sudo -i -u postgres


psql -p 5445 -d velbd -c "SHOW shared_buffers;"
psql -p 5445 -d velbd -c "SHOW work_mem;"
psql -p 5445 -d velbd -c "SHOW maintenance_work_mem;"
psql -p 5445 -d velbd -c "SHOW effective_cache_size;"


------------------------------------------------------------
admin1@debian-post:~$ sudo -i -u postgres
postgres@debian-post:~$ psql -p 5445 -d velbd -c "SHOW shared_buffers;"
psql -p 5445 -d velbd -c "SHOW work_mem;"
psql -p 5445 -d velbd -c "SHOW maintenance_work_mem;"
psql -p 5445 -d velbd -c "SHOW effective_cache_size;"
 shared_buffers
----------------
 2GB
(1 row)

 work_mem
----------
 32MB
(1 row)

 maintenance_work_mem
----------------------
 1GB
(1 row)

 effective_cache_size
----------------------
 7GB
(1 row)

postgres@debian-post:~$
--------------------------------------------------------------


почему не использовал уже созданное
Да, public.clients подойдёт, но прямо сейчас — не очень показательно для пункта про индексы, потому что:

в ней, судя по структуре, обычно мало строк (1–1000), и разницы “до/после” почти не будет;

индексы уже есть: PRIMARY KEY (id) и UNIQUE (phone) — значит для запросов по id и phone оптимизатор и так будет использовать индекс, и ты не сможешь честно показать “до индекса был Seq Scan, после стал Index Scan”.



psql -p 5445 -d velbd


CREATE TABLE IF NOT EXISTS public.sales (
  id bigserial PRIMARY KEY,
  customer_id int NOT NULL,
  product_id int NOT NULL,
  price numeric(10,2) NOT NULL,
  created_at timestamp NOT NULL DEFAULT now()
);


velbd=# ANALYZE public.sales;
ANALYZE


EXPLAIN ANALYZE
SELECT * FROM public.sales
WHERE customer_id = 4242;


------------------
velbd=# EXPLAIN ANALYZE
SELECT * FROM public.sales
WHERE customer_id = 4242;
                                                    QUERY PLAN
-------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..7281.77 rows=6 width=30) (actual time=13.719..21.807 rows=3 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on sales  (cost=0.00..6281.17 rows=2 width=30) (actual time=8.197..11.850 rows=1 loops=3)
         Filter: (customer_id = 4242)
         Rows Removed by Filter: 166666
 Planning Time: 0.139 ms
 Execution Time: 21.844 ms
(8 rows)

velbd=#
------------------------------------------------------------------------------------------------------------------


CREATE INDEX idx_sales_customer_id ON public.sales(customer_id);

EXPLAIN ANALYZE
SELECT * FROM public.sales
WHERE customer_id = 4242;


                                                          QUERY PLAN                                                    
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on sales  (cost=4.47..27.82 rows=6 width=30) (actual time=0.078..0.083 rows=3 loops=1)
   Recheck Cond: (customer_id = 4242)
   Heap Blocks: exact=3
   ->  Bitmap Index Scan on idx_sales_customer_id  (cost=0.00..4.47 rows=6 width=0) (actual time=0.072..0.072 rows=3 loops=1)
         Index Cond: (customer_id = 4242)
 Planning Time: 0.243 ms
 Execution Time: 0.163 ms
(7 rows)
------------------------------------------------------------------------------------------------------------------------------


CREATE TABLE IF NOT EXISTS public.payments (
  id bigserial PRIMARY KEY,
  amount numeric(10,2) NOT NULL,
  created_at timestamp NOT NULL DEFAULT now()
);


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

velbd=# SELECT public.add_payment(100.50);
SELECT public.add_payment(-10);

SELECT * FROM public.payments ORDER BY id DESC LIMIT 5;
   add_payment
------------------
 Запись добавлена
(1 row)

          add_payment
--------------------------------
 Ошибка: отрицательное значение
(1 row)

 id | amount |         created_at
----+--------+----------------------------
  1 | 100.50 | 2026-03-04 07:30:41.486048
(1 row)

velbd=#



CREATE TABLE IF NOT EXISTS public.products (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  price numeric(10,2) NOT NULL
);


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


DROP TRIGGER IF EXISTS check_price_before_ins_upd ON public.products;

CREATE TRIGGER check_price_before_ins_upd
BEFORE INSERT OR UPDATE ON public.products
FOR EACH ROW
EXECUTE FUNCTION public.trg_check_price();



INSERT INTO public.products(name, price) VALUES ('Phone', 500);
INSERT INTO public.products(name, price) VALUES ('BadItem', -1); -- должно упасть

UPDATE public.products SET price = -10 WHERE name='Phone'; -- тоже должно упасть



velbd=# SHOW autovacuum;
 autovacuum
------------
 on
(1 row)



velbd=# SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_scale_factor;
 autovacuum_naptime
--------------------
 1min
(1 row)

 autovacuum_vacuum_scale_factor
--------------------------------
 0.2
(1 row)

 autovacuum_analyze_scale_factor
---------------------------------
 0.1
(1 row)

velbd=#



VACUUM (ANALYZE) public.sales;


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

-------------------------------------------------------------------------------------------------------------
 relname  | n_live_tup | n_dead_tup | vacuum_count | autovacuum_count | analyze_count | autoanalyze_count |          last_vacuum          |        last_autovacuum        |         last_analyze          | last_autoanalyze
----------+------------+------------+--------------+------------------+---------------+-------------------+-------------------------------+-------------------------------+-------------------------------+------------------
 clients  |          2 |          4 |            0 |                0 |             0 |                 0 |                               |                               |                               |
 clothes  |          2 |          0 |            0 |                0 |             0 |                 0 |                               |                               |                               |
 payments |          1 |          0 |            0 |                0 |             0 |                 0 |                               |                               |                               |
 sales    |     500000 |          0 |            1 |                1 |             2 |                 0 | 2026-03-04 07:33:48.364341-05 | 2026-02-14 11:35:56.049996-05 | 2026-03-04 07:33:48.415086-05 |
 products |          1 |          0 |            0 |                0 |             0 |                 0 |                               |                               |                               |
 orders   |          2 |          0 |            0 |                0 |             0 |                 0 |                               |                               |                               |
(6 rows)

(END)
---------------------------------------------------------------------------------------------------------------

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



----------------------------------------------------------------------------------------------------------------
 schemaname  | relname  |     indexrelname      | idx_scan | idx_tup_read | idx_tup_fetch
-------------+----------+-----------------------+----------+--------------+---------------
 public      | clients  | clients_pkey          |        3 |            3 |             1
 public      | sales    | idx_sales_customer_id |        1 |            3 |             0
 public      | payments | payments_pkey         |        1 |            1 |             1
 test_schema | clothes  | clothes_pkey          |        0 |            0 |             0
 public      | sales    | sales_pkey            |        0 |            0 |             0
 public      | products | products_pkey         |        0 |            0 |             0
 public      | clients  | clients_phone_key     |        0 |            0 |             0
 test_schema | orders   | orders_pkey           |        0 |            0 |             0
(8 rows)

velbd=#
----------------------------------------------------------------------------------------------------------------








