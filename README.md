# psql
Репликация Postgres

    Vagranfile (2 машины)
    плейбук Ansible
    конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,
    конфиг barman, либо скрипт резервного копирования.

Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием. Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.

Пример плейбука:

    name: Установка postgres11
    hosts: master, slave
    become: yes
    roles:
        postgres_install

    name: Настройка master
    hosts: master
    become: yes
    roles:
        master-setup

    name: Настройка slave
    hosts: slave
    become: yes
    roles:
        slave-setup

    name: Создание тестовой БД
    hosts: master
    become: yes
    roles:
        create_test_db

    name: Настройка barman
    hosts: barman
    become: yes
    roles:
        barman_install tags:
        barman

Исполнение
Поднятие виртуалок

cd ../../
cd ./040/vm/
vagrant destroy -f
vagrant up
python3 v2a.py -o ../ansible/inventories/hosts # Это уже как кредо
cd ../../
cd ./040/ansible/

Репликация

    https://habr.com/ru/post/213409/

Hot standby — позволяет slave нодам обслуживать READ запросы для балансировки нагрузки, в отличии от warm standby, при котором slave сервер не обслуживает запросов клиентов, а только постоянно подтягивает себе с мастера актуальную базу. 

Настройка мастера
см. pg_hba.conf
см. postgresql.conf

ansible-playbook playbooks/master.yml --tags deploy > ../files/001_playbooks-master.yml.txt

см. лог выполнения `playbooks/master.yml`
Настройка реплики
см. postgresql.conf

ansible-playbook playbooks/replica.yml --tags deploy > ../files/002_playbooks-replica.yml.txt

см. лог выполнения `playbooks/replica.yml`
Проверка работоспособности
Что есть на реплике до какой-либо CRUD-"активности" на мастере

ansible-playbook playbooks/replica_check_before.yml > ../files/003_playbooks-replica_check_before.yml.txt

см. лог выполнения `playbooks/replica_check_before.yml`

Полученные результаты:
см. результат `SELECT datname AS databases FROM pg_database`
см. результат `SELECT schema_name FROM information_schema.schemata`
см. результат `SELECT schemaname, tablename FROM pg_catalog.pg_tables`
Что есть на мастере и совершаем CRUD-"действия"

ansible-playbook playbooks/master_check_and_activity.yml > ../files/004_playbooks-master_check_and_activity.yml.txt

см. лог выполнения `playbooks/master_check_and_activity.yml`
... что есть на мастере
см. результат `SELECT application_name, state, sync_priority, sync_state FROM pg_stat_replication`
см. результат `SELECT * FROM pg_stat_replication`

Полученные ниже результаты пригодятся для сравнения с изменениями на реплике после CRUD-"активности" на мастере:
см. результат `SELECT datname AS databases FROM pg_database`
см. результат `SELECT schema_name FROM information_schema.schemata`
см. результат `SELECT schemaname, tablename FROM pg_catalog.pg_tables`
... совершаем CRUD-"действия" на мастере
см. лог выполнения `DROP SCHEMA IF EXISTS test_schema CASCADE ... CREATE SCHEMA test_schema`
см. лог выполнения `CREATE TABLE test_schema.test_table ...`
см. лог выполнения `INSERT INTO test_schema.test_table VALUES ...`
Что есть на реплике после CRUD-"активности" на мастере

ansible-playbook playbooks/replica_check_after.yml > ../files/005_playbooks-replica_check_after.yml.txt

см. лог выполнения `playbooks/replica_check_after.yml`

Базы данных не изменились, так как не создавалось новых:
см. результат `SELECT datname AS databases FROM pg_database`

Появились сведения о созданной на мастере схеме test_schema:
см. результат `SELECT schema_name FROM information_schema.schemata`

Появились сведения о созданной на мастере таблице test_table схемы test_schema:
см. результат `SELECT schemaname, tablename FROM pg_catalog.pg_tables`

Появились реплицированные данные от мастера с таблицы test_table схемы test_schema:
см. результат `SELECT * FROM test_schema.test_table.txt`
Barman
Вводная

Barman сначала у меня не заработал до конца. Вопросы в самом конце. Я пробовал на PG11 и PG13. Сейчас репозиторий приведен под PG13. Есть отличие в реплицировании, так как было упразднено в PG12 recovery.conf, а его содержание перенесено в postgresql.conf. Вместо этого recovery.signal и standby.signal. Вместо trigger_file переименовали в promote_trigger_file. Если не инетересно все это исследование, то просто перейдите к разделу Barman - сухое изложение
Дополнительно на мастере

ansible-playbook playbooks/barman_on_master.yml --tags deploy > ../files/010_playbooks-barman_on_master.yml.txt

см. лог выполнения `playbooks/barman_on_master.yml`

Промежуточная проверка. Соединения локально работают:
см. PSQL: barman SELECT version()
см. PSQL: streaming_barman IDENTIFY_SYSTEM
На Barman-сервере

Я решил использовать вариант streaming режима (то есть не Rsync via SSH).

Подробная официальная документация:

    https://docs.pgbarman.org/release/2.15/

Есть еще:

    https://medium.com/coderbunker/implement-backup-with-barman-bb0b44af71f9
    https://blog.dbi-services.com/postgresql-barman-rsync-method-vs-streaming-method/
    https://sidmid.ru/barman-%D0%BC%D0%B5%D0%BD%D0%B5%D0%B4%D0%B6%D0%B5%D1%80-%D0%B1%D1%8D%D0%BA%D0%B0%D0%BF%D0%BE%D0%B2-%D0%B4%D0%BB%D1%8F-%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80%D0%BE%D0%B2-postgresql/
    https://habr.com/ru/company/yoomoney/blog/333844/
    http://innerlife.io/ru/fault-tolerant-postgresql-cluster-part4-2/
    https://virtbox.blogspot.com/2013/11/barman-postgresql-backup-and-recovery.html
    https://oguridatabasetech.com/2018/02/06/barman-error-impossible-to-start-the-backup-check-the-log-for-more-details/
    https://postgrespro.ru/list/thread-id/2371354
    https://itectec.com/database/postgresql-error-receiving-wal-files-with-barman/

Это все привожу потому, что у меня НЕ ВЫХОДИТ получить бекап и наладить передачу WAL.

Подробно.

ansible-playbook playbooks/barman_on_backup.yml --tags install-postgresql > ../files/011_playbooks-barman_on_backup-install-postgresql.yml.txt

см. лог выполнения `playbooks/barman_on_backup.yml --tags install-postgresql`

Кстати: у Barman в зависимостях именно сервер PostgreSQL, одним клиентом psql не обойтись.

Показано, что соединения удаленно тоже работают:
см. PSQL: barman SELECT version()
см. PSQL: barman streaming_barman IDENTIFY_SYSTEM

Установим и настроим барман
см. общий конфиг `barman.conf`
см. конфиг подопечного сервера `pg.conf`

ansible-playbook playbooks/barman_on_backup.yml --tags install-and-configure-barman > ../files/012_playbooks-barman_on_backup-install-and-configure-barman.yml.txt

см. лог выполнения `playbooks/barman_on_backup.yml --tags install-and-configure-barman`

Сервис barman-cron

Замечание: тут Unit-user может быть barman.
см. код `barman-cron.service`
см. код `barman-cron.timer`

работает:
см. лог работы `barman-cron.service`

Но в ходе выполнения есть ошибки

"stderr_lines": [
    "ERROR: The WAL file 000000010000000000000004 has not been received in 30 seconds"
],
"stdout_lines": [
    "The WAL file 000000010000000000000004 has been closed on server 'pg'",
    "Waiting for the WAL file 000000010000000000000006 from server 'pg' (max: 30 seconds)"
]

Смотрим barman check pg

ansible-playbook playbooks/barman_on_backup.yml --tags barman-check > ../files/013_barman-check-001.txt

прошел c ошибками
см. лог выполнения `barman-check`

выдержка:

barman check pg | grep FAILED

    WAL archive: FAILED (please make sure WAL shipping is setup)
    replication slot: FAILED (slot 'barman' not initialised: is 'receive-wal' running?)
    backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
    minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
    receive-wal running: FAILED (See the Barman log file for more details)

Дадим полезную нагрузку WAL:

ansible-playbook playbooks/master_activity.yml > ../files/014_playbooks-master_activity.yml.txt

см. лог выполнения `playbooks/master_activity.yml`

Повторим отдельно принудительный сбор

ansible-playbook playbooks/barman_on_backup.yml --tags barman-force-switch-wal > ../files/015_barman-force-switch-wal-001.txt

см. лог выполнения `barman-force-switch-wal`

Как видим, есть ошибки

"stderr_lines": [
    "ERROR: The WAL file 000000010000000000000006 has not been received in 30 seconds"
],
"stdout_lines": [
    "The WAL file 000000010000000000000006 has been closed on server 'pg'",
    "Waiting for the WAL file 000000010000000000000006 from server 'pg' (max: 30 seconds)"
]

И ситуация не поменялась.

А вот дальше - МАГИЯ.

Запускаем barman cron - хотя это-то уже и так запущено в сервис-службе и по таймеру, вот это выше было
см. лог работы `barman-cron.service`

Итак, принудительно barman cron

ansible-playbook playbooks/barman_on_backup.yml --tags barman-cron

и повторим забор wal (тоже самое делалось и ранее в ходе --tags install-and-configure-barman)

ansible-playbook playbooks/barman_on_backup.yml --tags barman-force-switch-wal > ../files/016_barman-force-switch-wal-002.txt

O, чудо. Ошибки исполнения barman switch-wal --force --archive pg исчезли
см. лог выполнения `barman-force-switch-wal`

Выдержка:

The WAL file t000000010000000000000006 has been closed on server 'pg'
Waiting for the WAL file t000000010000000000000006 from server 'pg' (max: 30 seconds)
Processing xlog segments from streaming for pg
t000000010000000000000006

При этом и в чеке теперь нет обшибок с WAL (это также сохранится и при перезагрузке системы, будет FAILED и потом самодиагностируется в OK)

ansible-playbook playbooks/barman_on_backup.yml --tags barman-check > ../files/017_barman-check-002.txt

см. лог выполнения `barman-check`

Осталось FAILED только связанное с бекапом

Server pg:
    ...
    backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
    ...
    minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
    ...

Кроме того, все эти действия не позволяют делать backup, точнее он просто виснет

sudo su - barman

barman backup pg
    Starting backup using postgres method for server pg in /var/lib/barman/pg/base/20211030T001134
    Backup start at LSN: 0/7000060 (000000010000000000000007, 00000060)
    Starting backup copy via pg_basebackup for 20211030T001134

... и долгая-долгая тишина... вроде как, ничего не происходит.

но если посмотреть в параллельные процессы, то видно

barman backup pg &
    [1] 22848
    -bash-4.2$ Starting backup using postgres method for server pg in /var/lib/barman/pg/base/20211120T121503
    Backup start at LSN: 0/9000148 (000000010000000000000009, 00000148)
    Starting backup copy via pg_basebackup for 20211120T184106 <----.
                                                                    |
                                                                    |
barman list-backup pg                                               |
    pg 20211120T184106 - STARTED <-------- вроде норм --------------'

ps ux | grep backup

     barman    3545  2.6  8.1 263540 19688 pts/0    S    09:09   0:00 /usr/bin/python2 /bin/barman backup pg
 --> barman    3548  3.3  1.6 180800  3864 pts/0    S    09:09   0:00 /bin/pg_basebackup --dbname=dbname=replication host=192.168.40.10 options=-cdatestyle=iso replication=true user=streaming_barman application_name=barman_streaming_backup -v --no-password --pgdata=/var/lib/barman/pg/base/20211120T090931/data --no-slot --wal-method=none --checkpoint=fast
     barman    3553  0.0  0.4 112812   976 pts/0    S+   09:09   0:00 grep --color=auto backup

Висит. Ничего не происходит.

И очень смущают аргументы без кавычек

/bin/pg_basebackup --dbname=dbname=replication host=192.168.40.10 options=-cdatestyle=iso replication=true user=streaming_barman application_name=barman_streaming_backup -v --no-password --pgdata=/var/lib/barman/pg/base/20211120T090931/data --no-slot --wal-method=none --checkpoint=fast

    pg_basebackup: error: too many command-line arguments (first is "host=192.168.40.10")
    Try "pg_basebackup --help" for more information.

Они должны быть такие

/bin/pg_basebackup --dbname='dbname=replication host=192.168.40.10 options=-cdatestyle=iso replication=true user=streaming_barman application_name=barman_streaming_backup' -v --no-password --pgdata=/var/lib/barman/pg/base/20211120T090931/data --no-slot --wal-method=none --checkpoint=fast

    pg_basebackup: initiating base backup, waiting for checkpoint to complete
    pg_basebackup: checkpoint completed
    NOTICE:  base backup done, waiting for required WAL segments to be archived
    WARNING:  still waiting for all required WAL segments to be archived (60 seconds elapsed)
    HINT:  Check that your archive_command is executing properly.  You can safely cancel this backup, but the database backup will not be usable without all the WAL segments.
    WARNING:  still waiting for all required WAL segments to be archived (120 seconds elapsed)
    HINT:  Check that your archive_command is executing properly.  You can safely cancel this backup, but the database backup will not be usable without all the WAL segments.

При этом
см. SQL: `SHOW archive_command`
см. SQL: `SHOW archive_mode`
Вот тут нужно добиться все же SSH

Так, смотрим тут https://docs.pgbarman.org/release/2.15/#wal-archiving-via-archive_command

From Barman 2.6, the recommended way to safely and reliably archive WAL files to Barman via archive_command is to use the barman-wal-archive command contained in the barman-cli package, distributed via EnterpriseDB public repositories and available under GNU GPL 3 licence. barman-cli must be installed on each PostgreSQL server that is part of the Barman cluster.

...

You can check that barman-wal-archive can connect to the Barman server, and that the required PostgreSQL server is configured in Barman to accept incoming WAL files with the following command:

barman-wal-archive --test backup pg DUMMY

исполнится успешно

[root@master vagrant]# barman-wal-archive --test backup pg DUMMY

При этом

[root@master vagrant]# cat /var/lib/pgsql/13/data/postgresql.conf | grep archive_command
# https://docs.pgbarman.org/release/2.15/#wal-archiving-via-archive_command
archive_command = 'barman-wal-archive backup pg %p'

[root@master vagrant]# cat /var/lib/pgsql/13/data/postgresql.conf | grep archive_mode
archive_mode = on

[root@master vagrant]# cat /var/lib/pgsql/13/data/postgresql.conf | grep wal_level
wal_level = replica # ВАЖНО В КОНТЕКСТЕ ЗАДАЧИ - replica

Осталось FAILED связанное с бекапом

barman check pg
Server pg:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
 -->    backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
 -->    minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK

barman  show-server pg
Server pg:
        active: True
        archive_timeout: 0
        archiver: False
        archiver_batch_size: 0
        backup_directory: /var/lib/barman/pg
        backup_method: postgres
        backup_options: BackupOptions(['concurrent_backup'])
        bandwidth_limit: None
        barman_home: /var/lib/barman
        barman_lock_directory: /var/lib/barman
        basebackup_retry_sleep: 30
        basebackup_retry_times: 0
        basebackups_directory: /var/lib/barman/pg/base
        check_timeout: 30
        checkpoint_timeout: 300
        compression: gzip
        config_file: /var/lib/pgsql/13/data/postgresql.conf
        connection_error: None
        conninfo: host=192.168.40.10 user=barman dbname=postgres
        create_slot: manual
        current_lsn: 0/D01CBF0
        current_size: 25572051
        current_xlog: 00000001000000000000000D
        custom_compression_filter: None
        custom_decompression_filter: None
        data_checksums: off
        data_directory: /var/lib/pgsql/13/data
        description: PostgreSQL Streaming Backup
        disabled: False
        errors_directory: /var/lib/barman/pg/errors
        forward_config_path: False
        has_backup_privileges: True
        hba_file: /var/lib/pgsql/13/data/pg_hba.conf
        hot_standby: on
        ident_file: /var/lib/pgsql/13/data/pg_ident.conf
        immediate_checkpoint: True
        incoming_wals_directory: /var/lib/barman/pg/incoming
        is_in_recovery: False
        is_superuser: True
        last_backup_maximum_age: 4 days (WARNING! latest backup is No available backups old)
        max_incoming_wals_queue: None
        max_replication_slots: 10
        max_wal_senders: 10
        minimum_redundancy: 1
        msg_list: []
        name: pg
        network_compression: False
        parallel_jobs: 1
        passive_node: False
        path_prefix: None
        pg_basebackup_bwlimit: True
        pg_basebackup_compatible: True
        pg_basebackup_installed: True
        pg_basebackup_path: /bin/pg_basebackup
        pg_basebackup_tbls_mapping: True
        pg_basebackup_version: 13.5
        pg_receivexlog_compatible: True
        pg_receivexlog_installed: True
        pg_receivexlog_path: /usr/pgsql-13/bin/pg_receivewal
        pg_receivexlog_supports_slots: True
        pg_receivexlog_synchronous: False
        pg_receivexlog_version: 13.5
        pgespresso_installed: False
        post_archive_retry_script: None
        post_archive_script: None
        post_backup_retry_script: None
        post_backup_script: None
        post_delete_retry_script: None
        post_delete_script: None
        post_recovery_retry_script: None
        post_recovery_script: None
        post_wal_delete_retry_script: None
        post_wal_delete_script: None
        postgres_systemid: 7032595315385409583
        pre_archive_retry_script: None
        pre_archive_script: None
        pre_backup_retry_script: None
        pre_backup_script: None
        pre_delete_retry_script: None
        pre_delete_script: None
        pre_recovery_retry_script: None
        pre_recovery_script: None
        pre_wal_delete_retry_script: None
        pre_wal_delete_script: None
        primary_ssh_command: None
        recovery_options: RecoveryOptions([])
        replication_slot: Record(slot_name='barman', active=True, restart_lsn='0/D000000')
        replication_slot_support: True
        retention_policy: REDUNDANCY 3
        retention_policy_mode: auto
        reuse_backup: None
        server_txt_version: 13.5
        slot_name: barman
        ssh_command: None
        streaming: True
        streaming_archiver: True
        streaming_archiver_batch_size: 0
        streaming_archiver_name: barman_receive_wal
        streaming_backup_name: barman_streaming_backup
        streaming_conninfo: host=192.168.40.10 user=streaming_barman dbname=postgres
        streaming_supported: True
        streaming_systemid: 7032595315385409583
        streaming_wals_directory: /var/lib/barman/pg/streaming
        synchronous_standby_names: ['standby']
        tablespace_bandwidth_limit: None
        timeline: 1
        wal_compression: off
        wal_keep_size: 0
        wal_level: replica
        wal_retention_policy: MAIN
        wals_directory: /var/lib/barman/pg/wals
        xlog_segment_size: 16777216
        xlogpos: 0/D01CBF0

На бекапе

[root@backup vagrant]# barman show-server pg | grep incoming_wals_directory
        incoming_wals_directory: /var/lib/barman/pg/incoming

На маcтере папка такая /var/lib/pgsql/13/data/pg_wal/ и команда на нем

[vagrant@master ~]$ sudo cat /var/lib/pgsql/13/data/postgresql.conf | grep archive_command
# https://docs.pgbarman.org/release/2.15/#wal-archiving-via-archive_command
archive_command = 'barman-wal-archive backup pg %p'

[vagrant@master ~]$ barman-wal-archive backup pg /var/lib/pgsql/13/data/pg_wal
ERROR: Error executing ssh: [Errno 13] Permission denied: '/var/lib/pgsql/13/data/pg_wal'

[vagrant@master ~]$ sudo su

[root@master vagrant]# barman-wal-archive backup pg /var/lib/pgsql/13/data/pg_wal
ERROR: WAL_PATH cannot be a directory: /var/lib/pgsql/13/data/pg_wal

[root@master vagrant]# barman-wal-archive backup pg /var/lib/pgsql/13/data/pg_wal/000000010000000000000001 
ERROR: Error executing ssh: [Errno 32] Broken pipe
Exception ValueError: 'I/O operation on closed file' in <bound method _Stream.__del__ of <tarfile._Stream instance at 0x7fbed9d6c638>> ignored

Тут меня осенило

ERROR: Error executing ssh: [Errno 32] Broken pipe <--- SSH

В документации https://docs.pgbarman.org/release/2.15/#two-typical-scenarios-for-backups в сценарии 1a не нужен SSH при STREAMING

This setup, in Barman's terminology, is known as **streaming-only** setup, as it does not require any SSH connection for backup and archiving operations. 

Но у тут же сценарий 1b https://docs.pgbarman.org/release/2.15/#two-typical-scenarios-for-backups

This alternate approach requires:

    an additional SSH connection that allows the postgres user on the PostgreSQL server to connect as barman user on the Barman server
    the archive_command in PostgreSQL be configured to ship WAL files to Barman

barman-wal-archive 192.168.40.12 pg /var/lib/pgsql/13/data/pg_wal/000000010000000000000010

Настроил SSH на MASTER-сервере для пользователя postgresql

и SSH на BACKUP-сервере для пользователя barman

и бекапирование пошло

-bash-4.2$ barman backup pg
Starting backup using postgres method for server pg in /var/lib/barman/pg/base/20211120T231221
Backup start at LSN: 0/350001C0 (000000010000000000000035, 000001C0)
Starting backup copy via pg_basebackup for 20211120T231221
Copy done (time: 6 seconds)
Finalising the backup.
This is the first backup for server pg
WAL segments preceding the current backup have been found:
        00000001000000000000002D from server pg has been removed
        00000001000000000000002E from server pg has been removed
        00000001000000000000002F from server pg has been removed
        000000010000000000000030 from server pg has been removed
        000000010000000000000031 from server pg has been removed
        000000010000000000000032 from server pg has been removed
        000000010000000000000033 from server pg has been removed
        000000010000000000000034 from server pg has been removed
Backup size: 24.6 MiB
Backup end at LSN: 0/37000060 (000000010000000000000037, 00000060)
Backup completed (start time: 2021-11-20 23:12:21.653246, elapsed time: 7 seconds)
Processing xlog segments from streaming for pg
        000000010000000000000035
        000000010000000000000036
        000000010000000000000037
 barman check pg

-bash-4.2$ 
Server pg:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: OK (interval provided: 4 days, latest backup age: 10 seconds) <----- WOW
        compression settings: OK
        failed backups: FAILED (there are 8 failed backups)       <----- Горький опыт
        minimum redundancy requirements: OK (have 5 backups, expected at least 1)  <----- WOW
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archive_mode: OK
        archive_command: OK
        continuous archiving: OK
        archiver errors: OK

Не понимаю, почему нужно barman cron запустить не в сервисе systemd по расписанию изначально, а просто в терминале. Так же пробовал изменить подход запуска служб по расписанию и использовать привычный crontab.

- name: Cron /usr/bin/barman backup
  become_user: barman
  cron:
    name: "barman backup pg"
    minute: "15,45"
    job: "/usr/bin/barman  backup pg"
  tags:
    - crontab

- name: Cron /usr/bin/barman cron
  become_user: barman
  cron:
    name: "barman cron"
    minute: '*'
    job: "/usr/bin/barman cron"
  tags:
    - crontab

как выглядит crontab

$ crontab -l

#Ansible: /usr/bin/barman backup pg
15,45 * * * * /usr/bin/barman  backup pg
#Ansible: /usr/bin/barman cron
* * * * * /usr/bin/barman cron

Barman - сухое изложение

Представим что виртуалки бекапа и мастера еще не "изменялись" (на самом деле у меня есть там конструкция с --tags redeploy или --tags clean, но это только затруднит понимание)
Настройка мастера для будущего Barman на backup-сервере

ansible-playbook playbooks/barman_on_master.yml --tags feature-deploy > ../files/110_playbooks-barman_on_master.yml.txt

см. лог выполнения `playbooks/barman_on_master.yml`
На backup-сервере

ansible-playbook playbooks/barman_on_backup.yml --tags feature-deploy > ../files/111_playbooks-barman_on_backup-feature_deploy.yml.txt

см. лог выполнения `playbooks/barman_on_backup.yml --tags feature_deploy`

Как видим выше, есть FAILED.

НО!

Но стоит выполнить barman cron и их часть исчезнет, хотя при этом barman cron уже итак работает в crontab пользователя barman.
см. лог выполнения `barman cron`

Исчезнут 2 FAILED
см. лог выполнения `barman check pg`
см. лог выполнения `barman backup pg`
см. лог выполнения `barman list-backup pg`

Исчезнут FAILED, связанные с бекапированием
см. лог выполнения `barman check pg`

Эти последние команды можно выполнить ролью

ansible-playbook playbooks/barman_on_backup.yml --tags kick-barman > ../files/114_playbooks-barman_on_backup-kick_barman.yml.txt

см. лог выполнения `playbooks/barman_on_backup.yml --tags kick-barman`

Если перезапустить виртуалку, то опять понадобиться сделать barman cron. Это единственное, что я не могу понять, почему он не рабтает в службе или по крону. хотя лог запуска и там и там есть.
Разный шлак мне на память, но не в ДЗ

cd ../../
cd ./040/vm/
vagrant destroy -f
vagrant up
python3 v2a.py -o ../ansible/inventories/hosts # Это уже как кредо
cd ../../
cd ./040/ansible/

ansible-playbook playbooks/master.yml --tags deploy > ../files/001_playbooks-master.yml.txt

ansible-playbook playbooks/replica.yml --tags deploy > ../files/002_playbooks-replica.yml.txt

ansible-playbook playbooks/replica_check_before.yml > ../files/003_playbooks-replica_check_before.yml.txt

ansible-playbook playbooks/master_check_and_activity.yml > ../files/004_playbooks-master_check_and_activity.yml.txt

ansible-playbook playbooks/replica_check_after.yml > ../files/005_playbooks-replica_check_after.yml.txt

ansible-playbook playbooks/barman_on_master.yml --tags feature-deploy > ../files/110_playbooks-barman_on_master.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags feature-deploy > ../files/111_playbooks-barman_on_backup-feature_deploy.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags kick-barman > ../files/114_playbooks-barman_on_backup-kick_barman.yml.txt








ansible-playbook playbooks/barman_on_master.yml --tags deploy > ../files/010_playbooks-barman_on_master.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags install-postgresql > ../files/011_playbooks-barman_on_backup-install-postgresql.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags install-and-configure-barman > ../files/012_playbooks-barman_on_backup-install-and-configure-barman.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags barman-check > ../files/013_barman-check-001.txt

ansible-playbook playbooks/master_activity.yml > ../files/014_playbooks-master_activity.yml.txt

ansible-playbook playbooks/barman_on_backup.yml --tags barman-force-switch-wal > ../files/015_barman-force-switch-wal-001.txt

ansible-playbook playbooks/barman_on_backup.yml --tags barman-cron

ansible-playbook playbooks/barman_on_backup.yml --tags barman-force-switch-wal > ../files/016_barman-force-switch-wal-002.txt

ansible-playbook playbooks/barman_on_backup.yml --tags barman-check > ../files/017_barman-check-002.txt









* * * * * export PATH=$PATH:/usr/pgsql-9.6/bin; barman cron >/dev/null 2>&1





barman check pg

cd ../../
./details.py 040.details.md 0

cd ../../
cd ./040/vm/
vagrant destroy -f
vagrant up
python3 v2a.py -o ../ansible/inventories/hosts # Это уже как кредо
cd ../../
cd ./040/ansible/

ansible-playbook playbooks/master.yml --tags deploy
ansible-playbook playbooks/replica.yml --tags deploy

ansible-playbook playbooks/replica_check_before.yml 
ansible-playbook playbooks/master_check_and_activity.yml 
ansible-playbook playbooks/replica_check_after.yml

ansible-playbook playbooks/barman_on_master.yml --tags deploy
 
ansible-playbook playbooks/barman_on_backup.yml --tags install-postgresql-repo
ansible-playbook playbooks/barman_on_backup.yml --tags install-postgresql-server
ansible-playbook playbooks/barman_on_backup.yml --tags remote-access-check
 
ansible-playbook playbooks/barman_on_backup.yml --tags install-epel-repo 
ansible-playbook playbooks/barman_on_backup.yml --tags yum-barman-requirements 
ansible-playbook playbooks/barman_on_backup.yml --tags install-barman-package
ansible-playbook playbooks/barman_on_backup.yml --tags copy-config-files
ansible-playbook playbooks/barman_on_backup.yml --tags create-barman-slot
ansible-playbook playbooks/barman_on_backup.yml --tags add-path
ansible-playbook playbooks/barman_on_backup.yml --tags barman-cron-service
ansible-playbook playbooks/master_activity.yml # полезная нагрузка WAL
#ansible-playbook playbooks/barman_on_backup.yml --tags barman-switch-wal
ansible-playbook playbooks/barman_on_backup.yml --tags barman-force-switch-wal


psql postgresql://192.168.40.10:5432/postgres?sslmode=require
psql -H 192.168.40.10:5432 -U postgres -W

# su - postgres
# psql template1
# CREATE USER replication WITH PASSWORD 'replication';
# GRANT ALL PRIVILEGES ON DATABASE "postgres" to replication;
# GRANT ALL PRIVILEGES ON DATABASE "postgres" to 'replication';
CREATE USER replication REPLICATION LOGIN CONNECTION LIMIT 5 ENCRYPTED PASSWORD 'ident';"

sudo -u postgres psql
CREATE DATABASE test;
CREATE USER test WITH ENCRYPTED PASSWORD 'test';
GRANT ALL PRIVILEGES ON DATABASE test TO test;
psql -h 192.168.40.10 -U test -d test 


CREATE USER replication REPLICATION LOGIN CONNECTION LIMIT 5 ENCRYPTED PASSWORD 'replication';"

ошибка: не удалось подключиться к серверу: Нет маршрута до узла
        Он действительно работает по адресу "192.168.40.10"
         и принимает TCP-соединения (порт 5432)?

netstat -nlp | grep 5432

systemctl stop postgresql-11
systemctl start postgresql-11
sudo service postgresql-11 restart

psql -h 192.168.40.10 -U replication -d replication -W
psql -h 192.168.40.10 -U test -d test -W

lsof -i | grep 'post'

Тогда вы можете узнать, какой порт слушает.
psql -U postgres -p "port_in_use"

ansible-playbook playbooks/master.yml   --tags collect-pg.conf-files
ansible-playbook playbooks/replica.yml  --tags collect-pg.conf-files
ansible-playbook playbooks/master.yml   --tags deploy
ansible-playbook playbooks/replica.yml  --tags install-epel-repo
ansible-playbook playbooks/replica.yml  --tags install-postgresql
ansible-playbook playbooks/replica.yml  --tags create-postgresql-data-dir
ansible-playbook playbooks/replica.yml  --tags install-python3-pip
ansible-playbook playbooks/replica.yml  --tags install-python3-pexpect
ansible-playbook playbooks/replica.yml  --tags copy-master-data
ansible-playbook playbooks/master.yml   --tags collect-pg.conf-files


SELECT datname FROM pg_database;
SELECT schema_name FROM information_schema.schemata;
SELECT schemaname, tablename FROM pg_catalog.pg_tables;
SELECT * FROM public.testtable;
\c test 

Заместки

    pexpect==3.3 - очень важно именно такая версия, так как в репе 2.*
    generic/centos7 - плохой дистрибутив для реплицирования PostgreSQL, что-то с недоступностью по 5432

ansible-playbook playbooks/barman_on_backup.yml --tags deploy ansible-playbook playbooks/barman_on_backup.yml --tags step-001 ansible-playbook playbooks/barman_on_backup.yml --tags precheck-barman ansible-playbook playbooks/barman_on_backup.yml --tags precheck-replicator-barman ansible-playbook playbooks/barman_on_backup.yml --tags install-barman ansible-playbook playbooks/barman_on_backup.yml --tags configure-barman ansible-playbook playbooks/barman_on_backup.yml --tags copy-config-files ansible-playbook playbooks/barman_on_master.yml ansible-playbook playbooks/barman_on_backup.yml --tags copy-config-files precheck_replicator_barman

su - barman barman receive-wal --create-slot pg barman switch-wal --force --archive pg barman check pg yum repolist yum repolist enabled

yum --disablerepo="*" --enablerepo='2ndquadrant-dl-default-release-pg11/7/x86_64' install barman

yum install postgresql11-libs.x86_64

Actually the given URL also says in the System requirements section:

Linux/Unix
Python >= 3.4
Python modules:
    argcomplete
    argh >= 0.21.2
    psycopg2 >= 2.4.2
    python-dateutil
    setuptools
PostgreSQL >= 8.3
rsync >= 3.0.4 (optional for PostgreSQL >= 9.2)

barman cron

ssh -o "StrictHostKeyChecking no" postgres@192.168.40.10

sudo yum -y install barman yum provides audit2allow

yum install policycoreutils-python

sudo audit2allow -a

                    echo $(date '+%Y-%m-%d') >> /var/lib/barman/1.txt

#============= sshd_t ============== allow sshd_t postgresql_db_t:file read;

sudo audit2allow -a -M pg_ssh sudo semodule -i pg_ssh.pp

#!!!! The file '/var/lib/pgsql/.ssh/authorized_keys' is mislabeled on your system.
#!!!! Fix with $ restorecon -R -v /var/lib/pgsql/.ssh/authorized_keys allow sshd_t postgresql_db_t:file open;

#!!!! This avc is allowed in the current policy allow sshd_t postgresql_db_t:file read;

#!!!! The file '/var/lib/pgsql/.ssh/authorized_keys' is mislabeled on your system.
#!!!! Fix with $ restorecon -R -v /var/lib/pgsql/.ssh/authorized_keys allow sshd_t postgresql_db_t:file getattr;

#!!!! This avc is allowed in the current policy allow sshd_t postgresql_db_t:file { open read }; [vagrant@master ~]$ sudo audit2allow -a -M pg_ssh

barman cron

                    export PATH=$PATH:/usr/pgsql-9.6/bin; barman cron >/dev/null 2>&1

sudo su - barman Last login: Вт окт 26 19:42:21 UTC 2021 on pts/0 -bash-4.2$ barman cron

barman backup pg --wait

sudo su - postgres -bash-4.2$ psql psql (11.13) Type "help" for help.

postgres=# select * from pg_replication_slots; slot_name | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn ---------------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+--------------------- pg_slot_replication | | physical | | | f | t | 21626 | | | 0/B01FA90 | (1 row)

                    export PATH=$PATH:/usr/pgsql-11/bin; barman cron >/dev/null 2>&1 barman switch-xlog --force --archive

66 export PATH=$PATH:/usr/pgsql-11/bin/ 67 echo $PATH 68 barman receive-wal pg 69 barman check pg 70 barman switch-wal --force --archive pg 71 barman receive-wal pg 72 barman switch-xlog --force --archive 73 barman switch-xlog --force --archive pg 74 barman check pg 75 history

30 23 * * * /usr/bin/barman backup clust_dvdrental

systemctl status barman-cron systemctl enable barman-cron.service systemctl status barman-cron.service systemctl status barman-cron.timer systemctl stop barman-cron.timer systemctl start barman-cron.timer sudo journalctl -u barman-cron

sudo su - barman barman receive-wal --create-slot pg barman cron barman switch-wal --force --archive pg barman backup pg barman check pg barman switch-wal pg

yum repolist yum repolist enabled

barman show-server pg barman check pg

barman backup pg &

barman list-backup pg pg 20211118T211728 - STARTED

2021-11-18 21:17:28,604 [22577] barman.backup_executor INFO: Starting backup copy via pg_basebackup for 20211118T211728 2021-11-18 21:17:28,959 [22086] barman.command_wrappers INFO: pg: pg_receivewal: finished segment at 0/13000000 (timeline 1) 2021-11-18 21:17:29,751 [22086] barman.command_wrappers INFO: pg: pg_receivewal: finished segment at 0/14000000 (timeline 1) 2021-11-18 21:18:02,705 [22591] barman.wal_archiver INFO: Found 2 xlog segments from streaming for pg. Archive all segments in one run. 2021-11-18 21:18:02,705 [22591] barman.wal_archiver INFO: Archiving segment 1 of 2 from streaming: pg/000000010000000000000012 2021-11-18 21:18:02,717 [22592] barman.server INFO: Another archive-wal process is already running on server pg. Skipping to the next server 2021-11-18 21:18:02,957 [22591] barman.wal_archiver INFO: Archiving segment 2 of 2 from streaming: pg/000000010000000000000013 2021-11-18 21:19:01,767 [22602] barman.wal_archiver INFO: No xlog segments found from streaming for pg. 2021-11-18 21:20:02,164 [22613] barman.wal_archiver INFO: No xlog segments found from streaming for pg. 2021-11-18 21:21:03,596 [22626] barman.wal_archiver INFO: No xlog segments found from streaming for pg.

barman backps aux | grep backup barman 22893 0.5 8.1 263540 19696 pts/0 S 21:41 0:00 /usr/bin/python2 /bin/barman backup pg barman 22896 0.7 1.6 180800 3860 pts/0 S 21:41 0:00 /bin/pg_basebackup --dbname=dbname=replication host=192.168.40.10 options=-cdatestyle=iso replication=true user=streaming_barman application_name=barman_streaming_backup -v --no-password --pgdata=/var/lib/barman/pg/base/20211118T214107/data --no-slot --wal-method=none --checkpoint=fast

barman list-server ansible-playbook playbooks/master.yml --tags deploy > ../files/001_playbooks-master.yml.txt
