### Обновление Cisco Catalyst
Обновить системное программное обеспечение Cisco Catalyst 2900 с сохранением текущей конфигурации и предыдущего ПО в качестве резервной копии на внешнем носителе.

Протестировано на следующих устройствах: ``Cisco Catalyst 2950/2960.``

Подключаемся через последовательный интерфейс к консоли управления Cisco.

После авторизации на маршрутизаторе получаем возможность исполнять команды в оболочке ``IOS``.
```bash
switch> enable
switch#
```
Проверяем наличие файловых систем на коммутаторе:
```bash
switch# show file system
```
Получаем список имеющихся на устройстве файловых систем с описанием параметров.

Используя полученные данные можно перемещаться по файловым системам и поискать файлы, подлежащие резервному копированию.

Применительно к "свежей", настроенной по умолчанию, Cisco Catalyst 2950 список файловых систем может выглядеть следующим образом:
```bash
File Systems:
Size(b) Free(b) Type Flags Prefixes
7741440 2863104 flash rw flash:
- - opaque ro bs:
32768 32716 nvram rw nvram:
- - opaque rw null:
- - opaque rw system:
- - network rw tftp:
- - opaque ro xmodem:
- - opaque ro ymodem:
- - network rw rcp:
- - network rw ftp:
- - opaque ro cns:
....
```
Сразу обращаем внимание на объем доступного пространства в основной файловой системе, его у нас не более трёх мегабайт, что не позволит разместить ещё один образ операционной системы для данного устройства.<br> 
Это означает то, придётся подвергнуть файловую систему зачистке для высвобождения пространства для последующей загрузки более свежего ПО.

Просматриваем содержимое файловой системы коммутатора:
```bash
switch# cd flash:
switch# dir
```
Видим там нечто вроде следующего:
```bash
Directory of flash:/
2 -rwx 109 Mar 01 1993 00:02:10 +00:00 info
3 -rwx 269 Jan 01 1970 00:02:07 +00:00 env_vars
8 -rwx 3117390 Mar 01 1993 00:03:49 +00:00 c2950-i6q4l2-mz.121-
22.EA8.bin
9 drwx 4096 Mar 01 1993 00:04:27 +00:00 html
14 -rwx 109 Mar 01 1993 00:05:05 +00:00 info.ver
7741440 bytes total (2863104 bytes free)
....
```
Так вот, ``"c2950-i6q4l2-mz.121-22.EA8.bin"``- это и есть образ IOS для Catalyst. Единственный вариант обновления IOS - использовать TFTP сервер.<br>
Развёртываем TFTP сервер для загрузки с него на устройство образа IOS. Для Windows не знаю ничего лучше TFTPD32 (http://tftpd32.jounin.net).<br>
Считаем, что мы имеем устройство Catalyst, подключённое к сети, в которой есть TFTP сервер (по адресу 192.168.1.200/24) с требуемым нам программным обеспечением. настроим один из сетевых интерфейсов:
```bash
switch# conf t
switch(config)# interface FastEthernet0/1
switch(config-if)# switchport access vlan 1
switch(config-if)# switchport mode access
switch(config-if)# exit
switch(config)# interface Vlan 1
switch(config-if)# ip address 192.168.1.1 255.255.255.0
switch(config-if)# no shutdown
switch(config-if)# exit
```
Копируем образ IOS и дистрибутивы дополнительного программного обеспечения на удалённый TFTP сервер:
```bash
switch# copy flash:/c2950-i6q4l2-mz.121-22.EA8.bin
tftp://192.168.1.200/c2950-i6q4l2-mz.121-22.EA8.bin
```
Удаляем образ IOS с устройства:
```bash
switch# delete flash:c2950-i6q4l2-mz.121-22.EA8.bin
```
Копируем образ с желаемым IOS для нашего устройства с удалённого TFTP сервера:
```bash
switch# copy tftp://192.168.1.200/c2950-i6k2l2q4-mz.121-22.EA13.bin
flash:/c2950-i6k2l2q4-mz.121-22.EA13.bin
```
Копируем резервную копию конфигурационного файла (если таковой имеется) устройства на удалённый сервер:
```bash
switch# copy nvram:startup-config tftp://192.168.1.200/
```
Процесс копирования сопровождается выводом в терминал символов "!". Один знак "!" соответствует десяти успешно скопированным пакетам.<br> 
После успешной загрузки файла будет произведено вычисление контрольной суммы файла.<br> 
Результат вычисления контрольной суммы очень неплохо бы сравнить с имеющимся у нас значением, полученным, например, при загрузке файла с сайта производителя.<br> 
Желательно так же принудительно вычислить контрольную сумму загруженного файла:
```bash
switch# verify /md5 flash:c2950-i6k2l2q4-mz.121-22.EA13.bin
```
Проверяем файловую систему на предмет наличия в ней загруженного файла:
```bash
switch# show flash:
```
Получаем нечто вроде следующего:
```bash
Directory of flash:/
....
4 -rwx 3721946 Mar 01 1993 00:52:23 +00:00 c2950-i6k2l2q4-mz.121-
22.EA13.bin
....
```
Укажем системе загружать новый образ (предварительно отключив загрузку предыдущего образа):
```bash
switch(config)# no boot system
switch(config)# boot system flash:/c2950-i6k2l2q4-mz.121-22.EA13.bin
switch(config)# exit
```
Убедится в том, что конфигурация загрузчика изменена верно можно следующим образом:
```bash
switch# show boot
BOOT path-list: flash:/c2950-i6k2l2q4-mz.121-22.EA13.bin
Config file: flash:/config.text
Private Config file: flash:/private-config.text
Enable Break: no
Manual Boot: no
HELPER path-list:
NVRAM/Config file
buffer size: 32768
```
Сохраняем параметры и перезапускаем устройство:
```bash
switch# copy running-config startup-config
switch# reload
```
После перезагрузки устройства проверяем, не были ли наши усилия тщетными и убеждаемся в обратном:
```bash
switch# show version
....
IOS (tm) C2950 Software (C2950-I6K2L2Q4-M), Version 12.1(22)EA13, RELEASE
SOFTWARE (fc2)
```
