sudo mkdir -p /var/backups/postgresql
sudo chown -R postgres:postgres /var/backups/postgresql
sudo chmod 750 /var/backups/postgresql

sudo -i -u postgres


pg_dump -p 5445 -U postgres -Fc -d velbd \
  -f /var/backups/postgresql/velbd_full_$(date +%F).dump

pg_dump -p 5445 -Fc -d velbd -n test_schema \
  -f /var/backups/postgresql/velbd_test_schema_$(date +%F).dump

pg_dump -p 5445 -Fc -d velbd -t test_schema.clothes   -f /var/backups/postgresql/velbd_public_ta
bles_$(date +%F).dump


createdb -p 5445 -O Vel velbd_restore

pg_restore -p 5445   -d velbd_restore   /var/backups/postgresql/velbd_full_2026-02-14.dump


psql -h 127.0.0.1 -p 5445 -U Vel -d velbd_restore -c "\dt"


exit

sudo nano /usr/local/bin/pg_backup_velbd.sh

------------------------
#!/usr/bin/env bash
set -euo pipefail

HOST="127.0.0.1"
PORT="5445"
DB="velbd"
USER="postgres"
BACKUP_DIR="/var/backups/postgresql"
KEEP_DAYS=7

DATE="$(date +%F)"
FILE="${BACKUP_DIR}/${DB}_full_${DATE}.dump"

# дамп в custom-формате
pg_dump -p "$PORT" -U "$USER" -Fc -d "$DB" -f "$FILE"

# ротация: удалить файлы старше KEEP_DAYS
find "$BACKUP_DIR" -type f -name "${DB}_full_*.dump" -mtime +"$KEEP_DAYS" -delete
---------------------------------------------------------------

sudo chmod +x /usr/local/bin/pg_backup_velbd.sh



sudo -i -u postgres


/usr/local/bin/pg_backup_velbd.sh
ls -lh /var/backups/postgresql


crontab -e


10 2 * * * /usr/local/bin/pg_backup_velbd.sh >> /var/log/pg_backup_velbd.log 2>&1


tail -n 50 /var/log/pg_backup_velbd.log


exit


psql -h 127.0.0.1 -p 5445 -U Vel -d velbd


SELECT pid, usename, datname, client_addr, state, wait_event_type, wait_event,
       now() - query_start AS running_for,
       left(query, 120) AS query
FROM pg_stat_activity
WHERE datname = 'velbd'
ORDER BY running_for DESC;


SELECT pid, usename, now() - query_start AS running_for, state,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '30 seconds'
ORDER BY running_for DESC;



SELECT pg_cancel_backend(<pid>);


SELECT pg_terminate_backend(<pid>);


sudo journalctl -u postgresql --no-pager -n 200


sudo ls -lah /var/log/postgresql/
sudo tail -n 200 /var/log/postgresql/postgresql-*.log


sudo ls -lah /var/log/
sudo tail -n 200 /var/log/syslog
sudo tail -n 200 /var/log/daemon.log
