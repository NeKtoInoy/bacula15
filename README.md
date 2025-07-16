# bacula15 ALTLinux
bacula15+baculum15 - postgresql17 - settings

### Настройка и  запуск postgresql

1. Установкка необходимых компонентов БД

```
apt-get install postgresql17-{server,contrib}
```

1. Инициализация postgresql

```
/etc/init.d/postgresql initdb
```

1. Настройка /var/lib/pgsql/data/postgresql.conf

```
wal_level = archive # указывается уровень ведения журналов
archive_mode = on # разрешается ведение журналов
archive_command = '/var/lib/pgsql/bin/copy_wal.sh "%f" "%p"'
```

1. Создание каталога для архива журналов и выдача необходимых прав

```
mkdir /var/lib/pgsql/wals
chown postgres:postgres /var/lib/pgsql/wals
chmod 700 /var/lib/pgsql/wals
```

1. Создание каталога для скрипта архивирующего журнал

```
mkdir /var/lib/pgsql/bin
chown postgres:postgres /var/lib/pgsql/bin
chmod 700 /var/lib/pgsql/bin
```

1. Сам скрипт /var/lib/pgsql/bin/copy_wal.sh

```
#!/bin/bash
DPATH=/var/lib/pgsql/wals
DATE=`date +"%b %d %T"`
if [ -e /var/lib/pgsql/backup_in_progress ]; then # По наличию файла проверяет не идет ли процесс резервного копирования журналов
 echo "${DATE} - идет процесс резервного копирования журналов" >> "${DPATH}/wal-c-log.log"
 exit 1
fi
if [ -e ${DPATH}/$1 ]; then # Проверяет скопирован ли журнал раннее
 echo "${DATE} - файл уже архивирован" >> "${DPATH}/wal-c-log.log"
 exit 1
fi
echo "${DATE} - /bin/gzip $2 ${DPATH}/$1" >> "${DPATH}/wal-c-log.log"
gzip < $2 > "${DPATH}/$1" # Архивирует файл журнала
```

1. Выдача необходимых прав на скрипт

```
chown postgres:postgres /var/lib/pgsql/bin/copy_wal.sh
chmod 700 /var/lib/pgsql/bin/copy_wal.sh
```

1. Запуск и добавление в автозапуск СУБД

```
systemctl enable --now postgresql.service
```

### Настройка серверной части bacula

1. Установка необходимых пакетов

   ```bash
   apt-get install bacula15-common bacula15-console bacula15-director-common bacula15-director-postgresql bacula15-storage mt-st -y
   ```
2. Задаем пароль для сккриптов bacula в файле /usr/share/bacula/scripts/grant_postgresql_privileges

   ```bash
   db_password="Ваш пароль"
   ```
3. Cоздание БД для bacula c помощью скрипта

   ```bash
   /usr/share/bacula/scripts/create_postgresql_database -U postgres
   ```
4. Создаем таблицы для БД  скриптом

   ```bash
   /usr/share/bacula/scripts/make_postgresql_tables -U postgres
   ```

::: error
Если не установить postgresql-contrib то тут посыпятся ошибки
:::

1. Установим права для пользователя bacula

   ```bash
   /usr/share/bacula/scripts/grant_postgresql_privileges -U postgres
   ```

### Настройка конфигурационных файлов для работы bacula

1. Задаем пароль для доступа диреектора к БД (/etc/bacula/bacula-dir.conf)

   ```bash
   ....
   Catalog {
     Name = MyCatalog
     dbname = bacula
     user = bacula
     password = "password"
   }
   ....
   ```
2.  задаем пароли в
   /etc/bacula/bacula-dir-password.conf
   /etc/bacula/bacula-fd-password.conf
3. Создаем конфигурационный файл для клиента в /etc/bacula/client.d/clien2.conf

   ```bash
   Client {
      Name = Client
      Address = 10.88.14.71
      FDPort = 9102
      Catalog = MyCatalog @/etc/bacula/bacula-fd-password.conf         File Retention = 30 days            # 30 days
      Job Retention = 6 months            # six months
      AutoPrune = yes
   }
   ```
4.  Настройка FileStorage (/etc/bacula/device.d/filestorage.conf)

   ```
   Device {
     Name = FileStorage # Имя устройства
     Media Type = File # Тип устройства
     Archive Device = /home/backup # Каталог для хранения
     LabelMedia = yes;                   # lets Bacula label unlabeled media
     Random Access = Yes;
     AutomaticMount = yes;               # when device opened, read it
     RemovableMedia = no;
     AlwaysOpen = no;
   }
   ```

Cоздание необходимой директории и выдача прав для нее

```bash
mkdir /home/backup
chown bacula:bacula /home/backup
chmod 777 -R /home/backup
```

В каталоге /etc/bacula/fileset.d находятся описания списков файлов для резервирования (задачи)

#### Можно заупскать bacula

```
systemctl enable --now bacula-dir
systemctl enable --now bacula-sd
systemctl enable --now bacula-fd
```

### Настройка клиентов

1. Скачать bacula-client

   ```
   apt-get install bacula15-client -y

   ```
2. Задать пароль в /etc/bacula/bacula-dir.password.conf и  /etc/bacula/bacula-fd.password.conf и  /etc/bacula/bacula-sd.password.conf
3. Включить и добавить в автозапуск bacula

   ```
   systemctl enable --now bacula-fd
   ```

## Для проверки зайти в bconsole и там

```
*status
Status available for:
     1: Director
     2: Storage
     3: Client
     4: Scheduled
     5: Network
     6: All
Select daemon type for status (1-6): 6
```

### Вывод должен быть похож на

```
dir Version: 15.0.2 (21 March 2024) x86_64-alt-linux-gnu redhat
Daemon started 03-ию-2025 12:47, conf reloaded 03-июн-2025 12:47:30
 Jobs: run=0, running=0 max=1 mode=0,0
 Crypto: fips=N/A crypto=OpenSSL 1.1.1w  11 Sep 2023
 Heap: heap=401,408 smbytes=334,686 max_bytes=334,686 bufs=376 max_bufs=381
 Res: njobs=3 nclients=2 nstores=1 npools=2 ncats=1 nfsets=2 nscheds=2

Scheduled Jobs (2/50):
Level          Type     Pri  Scheduled          Job Name           Volume
===================================================================================
Incremental    Backup    10  03-ию-2025 23:05 BackupFullSet      *unknown*
Full           Backup    11  03-ию-2025 23:10 BackupCatalog      *unknown*
====

Running Jobs:
Console connected using TLS at 03-ию-2025 12:57
No Jobs running.
====
No Terminated Jobs.
====
Connecting to Storage daemon File at 127.0.0.1:9103

sd Version: 15.0.2 (21 March 2024) x86_64-alt-linux-gnu redhat
Daemon started 03-ию-2025 12:47. Jobs: run=0, running=0 max=20.
 Ulimits: nofile=1024 memlock=8388608 status=nofile
 Heap: heap=274,432 smbytes=205,618 max_bytes=391,198 bufs=135 max_bufs=136
 Sizes: boffset_t=8 size_t=8 int32_t=4 int64_t=8 mode=0,0 newbsr=0
 Crypto: fips=N/A crypto=OpenSSL 1.1.1w  11 Sep 2023
 Res: ndevices=1 nautochgr=0
 Caps: APPEND_ONLY, IMMUTABLE

Running Jobs:
Director connected using TLS at: 03-ию-2025 12:57
No Jobs running.
====

Jobs waiting to reserve a drive:
====

Terminated Jobs:
====

Device status:

Device File: "FileStorage" (/home/backup) is not open.
   Available Space=27.53 GB
== but this not fully
====

Used Volume status:
====

====

Connecting to Client server at 127.0.0.1:9102

fd Version: 15.0.2 (21 March 2024)  x86_64-alt-linux-gnu redhat
Daemon started 03-ию-2025 12:48. Jobs: run=0 running=0 max=20.
 Heap: heap=270,336 smbytes=197,410 max_bytes=197,557 bufs=93 max_bufs=94
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Crypto: fips=N/A crypto=OpenSSL 1.1.1w  11 Sep 2023

Running Jobs:
Director connected using TLS at: 03-ию-2025 12:57
No Jobs running.
====

Terminated Jobs:
====
Connecting to Client Client at 10.88.14.71:9102

Client Version: 15.0.2 (21 March 2024)  x86_64-alt-linux-gnu redhat
Daemon started 04-ию-2025 11:35. Jobs: run=0 running=0 max=20.
 Heap: heap=270,336 smbytes=197,440 max_bytes=197,587 bufs=93 max_bufs=94
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Crypto: fips=N/A crypto=OpenSSL 1.1.1w  11 Sep 2023

Running Jobs:
Director connected using TLS at: 04-ию-2025 11:36
No Jobs running.
====

Terminated Jobs:
====
```

# Настройка Клиента
```

```
