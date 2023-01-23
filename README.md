# Практические навыки работы с ZFS
## Определить алгоритм с наилучшим сжатием.

Проверяем как поднялась VM из Vagrantfile

```
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 
```
Создаем 4 пула из 2-х дисков собранных в RAID-1

```
[root@zfs vagrant]# zpool create hw4-1 mirror /dev/sdb /dev/sdc
[root@zfs vagrant]# zpool create hw4-2 mirror /dev/sdd /dev/sde     
[root@zfs vagrant]# zpool create hw4-3 mirror /dev/sdf /dev/sdg
[root@zfs vagrant]# zpool create hw4-4 mirror /dev/sdh /dev/sdi
```
Проверяем
```
[root@zfs vagrant]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
hw4-1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
hw4-2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
hw4-3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
hw4-4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
Добавляем разные алгоритмы сжатия в каждую ФС

```
[root@zfs vagrant]# zfs set compression=lzjb hw4-1
[root@zfs vagrant]# zfs set compression=lz4 hw4-2
[root@zfs vagrant]# zfs set compression=gzip-9 hw4-3
[root@zfs vagrant]# zfs set compression=zle hw4-4
```
Проверим, что все выполнелось корректно

```
[root@zfs vagrant]# zfs get all | grep compression
hw4-1  compression           lzjb                   local
hw4-2  compression           lz4                    local
hw4-3  compression           gzip-9                 local
hw4-4  compression           zle                    local
```
Дальше циклом загружаем в каждый пул установочный пакет grafana-9.2.1-1.x86_64.rpm

```
ffor i in {1..4}; do wget -P /hw4-$i wget https://dl.grafana.com/oss/release/grafana-9.2.1-1.x86_64.rpm; done
```
Проверяем результат загрузки 

```
[root@zfs vagrant]# ls -l /hw4-*
/hw4-1:
total 93183
-rw-r--r--. 1 root root 95743787 Oct 18 11:17 grafana-9.2.1-1.x86_64.rpm

/hw4-2:
total 92952
-rw-r--r--. 1 root root 95743787 Oct 18 11:17 grafana-9.2.1-1.x86_64.rpm

/hw4-3:
total 92738
-rw-r--r--. 1 root root 95743787 Oct 18 11:17 grafana-9.2.1-1.x86_64.rpm

/hw4-4:
total 93503
-rw-r--r--. 1 root root 95743787 Oct 18 11:17 grafana-9.2.1-1.x86_64.rpm
```
Проверим, сколько места занимает один и тот же файл при разных методах сжатия (размер исходного файла 95,743,787)

```
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
hw4-1  91.1M   261M     91.0M  /hw4-1
hw4-2  90.9M   261M     90.8M  /hw4-2
hw4-3  90.7M   261M     90.6M  /hw4-3
hw4-4  91.5M   260M     91.3M  /hw4-4

[root@zfs ~]# du -s /hw4-1/
93184	/hw4-1/
[root@zfs ~]# du -s /hw4-2/
92952	/hw4-2/
[root@zfs ~]# du -s /hw4-3/
92738	/hw4-3/
[root@zfs ~]# du -s /hw4-4/
93503	/hw4-4/
```
Очевидно, что лучшая степень сжатия у алгоритма `gzip-9`.

## Определить настройки pool’a.

Загружаем указанный в ДЗ архив по ссылке на VM и распаковываем его

```
[root@zfs hw4]# tar -xf archive.tar 
[root@zfs hw4]# ls
archive.tar  zpoolexport
```
Проверям возможность импортирования
```
zpool import -d zpoolexport/
pool: имя пула 
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                             ONLINE
	  mirror-0                       ONLINE
	    /root/hw4/zpoolexport/filea  ONLINE
	    /root/hw4/zpoolexport/fileb  ONLINE

```
видим, что имя пула - otus, тип pool - mirror-0

Делаем импорт пула к в ОС:

```
[root@zfs hw4]# zpool import -d zpoolexport/ otus
[root@zfs hw4]# zpool status

...

pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                             STATE     READ WRITE CKSUM
	otus                             ONLINE       0     0     0
	  mirror-0                       ONLINE       0     0     0
	    /root/hw4/zpoolexport/filea  ONLINE       0     0     0
	    /root/hw4/zpoolexport/fileb  ONLINE       0     0     0

```
В выводе команды видим содержимое импортированного пула

Определим размер хранилища
```
[root@zfs hw4]# sudo zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
hw4-1   480M  91.2M   389M        -         -     2%    18%  1.00x    ONLINE  -
hw4-2   480M  91.0M   389M        -         -     2%    18%  1.00x    ONLINE  -
hw4-3   480M  90.7M   389M        -         -     2%    18%  1.00x    ONLINE  -
hw4-4   480M  91.5M   388M        -         -     3%    19%  1.00x    ONLINE  -
otus    480M  2.09M   478M        -         -     0%     0%  1.00x    ONLINE  -
```
Размер нашего хранилища - 480М

Определяем настройки импортированного пула

```
[root@zfs hw4]# zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default

```

Grep-нем нужные нам параметры

```
[root@zfs hw4]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
Значение recordsize - 128K

```
[root@zfs hw4]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
Используемое сжатие - zle

```
[root@zfs hw4]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
Используемая контрольная сумма - sha256

## Найти сообщение от преподавателей.

Качаем файл по ссылке из ДЗ

Восстанавливаем ФС из снапшота

```
[root@zfs wh4-5]# zfs receive -F hw4-2 < messege.file
[root@zfs wh4-5]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
hw4-1   480M  91.2M   389M        -         -     2%    18%  1.00x    ONLINE  -
hw4-2   480M  2.61M   477M        -         -     2%     0%  1.00x    ONLINE  -
hw4-3   480M  90.7M   389M        -         -     2%    18%  1.00x    ONLINE  -
hw4-4   480M  91.5M   388M        -         -     3%    19%  1.00x    ONLINE  -
otus    480M  2.09M   478M        -         -     0%     0%  1.00x    ONLINE  -
```
Проверяем

```
[root@zfs wh4-5]# ls -la /hw4-2/
total 2108
drwxr-xr-x.  3 root    root         11 May 15  2020 .
dr-xr-xr-x. 24 root    root       4096 Jan 22 18:04 ..
-rw-r--r--.  1 root    root          0 May 15  2020 10M.file
-rw-r--r--.  1 root    root     309987 May 15  2020 Limbo.txt
-rw-r--r--.  1 root    root     509836 May 15  2020 Moby_Dick.txt
-rw-r--r--.  1 root    root    1209374 May  6  2016 War_and_Peace.txt
-rw-r--r--.  1 root    root     727040 May 15  2020 cinderella.tar
-rw-r--r--.  1 root    root         65 May 15  2020 for_examaple.txt
-rw-r--r--.  1 root    root          0 May 15  2020 homework4.txt
drwxr-xr-x.  3 vagrant vagrant       4 Dec 18  2017 task1
-rw-r--r--.  1 root    root     398635 May 15  2020 world.sql
```
Далее, ищем в каталоге /otus/test файл с именем “secret_message”

```
[root@zfs wh4-5]# cd /hw4-2/
[root@zfs hw4-2]# ll
total 2102
-rw-r--r--. 1 root    root          0 May 15  2020 10M.file
-rw-r--r--. 1 root    root     309987 May 15  2020 Limbo.txt
-rw-r--r--. 1 root    root     509836 May 15  2020 Moby_Dick.txt
-rw-r--r--. 1 root    root    1209374 May  6  2016 War_and_Peace.txt
-rw-r--r--. 1 root    root     727040 May 15  2020 cinderella.tar
-rw-r--r--. 1 root    root         65 May 15  2020 for_examaple.txt
-rw-r--r--. 1 root    root          0 May 15  2020 homework4.txt
drwxr-xr-x. 3 vagrant vagrant       4 Dec 18  2017 task1
-rw-r--r--. 1 root    root     398635 May 15  2020 world.sql
[root@zfs hw4-2]# find -name "secret_message"
./task1/file_mess/secret_message
```
Смотрим содержимое найденного файла:

```
[root@zfs hw4-2]# cat /hw4-2/task1/file_mess/secret_message 

https://github.com/sindresorhus/awesome
```
Файл содержит следующую ссылку --> https://github.com/sindresorhus/awesome