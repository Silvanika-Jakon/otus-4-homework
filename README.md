# Домашнее задание: ZFS  
## Задание  
1. Определить алгоритм с наилучшим сжатием:  
  - определить, какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);  
  - создать 4 файловых системы, на каждой применить свой алгоритм сжатия;  
  - для сжатия использовать либо текстовый файл, либо группу файлов.  
2. Определить настройки пула.  
С помощью команды zfs import собрать pool ZFS.  
Командами zfs определить настройки:  
  - размер хранилища;  
  - тип pool;  
  - значение recordsize;  
  - какое сжатие используется;  
  - какая контрольная сумма используется.  
3. Работа со снапшотами:  
  - скопировать файл из удаленной директории;  
  - восстановить файл локально. zfs receive;  
  - найти зашифрованное сообщение в файле secret_message.  

## Выполнение:
### Задание 1.
Установим ZFS на систему  
````
apt install zfsutils-linux
```  

Создадим 4 zpool.  
```
root@efilimonova-infra:/home/user# zpool create otus1 mirror /dev/sdb /dev/sdc
root@efilimonova-infra:/home/user# zpool create otus2 mirror /dev/sdd /dev/sde
root@efilimonova-infra:/home/user# zpool create otus3 mirror /dev/sdf /dev/sdg
root@efilimonova-infra:/home/user# zpool create otus4 mirror /dev/sdh /dev/sdi
root@efilimonova-infra:/home/user# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   104K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   112K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   104K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   138K   480M        -         -     0%     0%  1.00x    ONLINE  -
root@efilimonova-infra:/home/user# zpool status
  pool: otus1
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors
```

Зададим для каждого отдельный вид сжатия.  
```
root@efilimonova-infra:/home/user# zfs set compression=lzjb otus1
root@efilimonova-infra:/home/user# zfs set compression=lz4 otus2
root@efilimonova-infra:/home/user# zfs set compression=gzip-9 otus3
root@efilimonova-infra:/home/user# zfs set compression=zle otus4
root@efilimonova-infra:/home/user# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```

Закачаем файл на каждый пул  
```
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2026-02-07 20:53:16--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
```

Видим, что самое эфеективное сжатие - gzip-9.  
```
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  21.7M   330M  21.6M  /otus1
otus2  17.7M   334M  17.6M  /otus2
otus3  10.9M   341M  10.7M  /otus3
otus4  39.5M   313M  39.4M  /otus4

zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.82x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.66x                  -
otus4  compressratio         1.00x                  -
```  

### Задание 2.  
Скачаем архив с zpool и распакуем. 
```
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAw                                                bNN1Bb&export=download'
...
tar -xzvf archive.tar.gz
```

Просмотрим информацию о полученном пуле:
```
 zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                              ONLINE
          mirror-0                        ONLINE
            /home/user/zpoolexport/filea  ONLINE
            /home/user/zpoolexport/fileb  ONLINE
```

Добавим его в нашу систему под именем otus.  
```
zpool import -d zpoolexport/ otus

zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                              STATE     READ WRITE CKSUM
        otus                              ONLINE       0     0     0
          mirror-0                        ONLINE       0     0     0
            /home/user/zpoolexport/filea  ONLINE       0     0     0
            /home/user/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors
```

Просмотрим важную информацию об этом пуле
```
root@efilimonova-infra:/home/user# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
root@efilimonova-infra:/home/user# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
root@efilimonova-infra:/home/user# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
root@efilimonova-infra:/home/user# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
root@efilimonova-infra:/home/user# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```

### Задание 3.  
Загрузим снапшот и восстановим его в файловую систему
```
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y0                                                29c3oI&export=download

zfs receive otus/test@today < otus_task2.file
```

Находим в примонтированном снапшоте нужный нам файл
```
find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```








