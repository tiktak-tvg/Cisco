### Базовая конфигурация Cisco Catalyst

базовая конфигурация Cisco Catalyst 2950/2960 в качестве подготовки для последующей
индивидуальной настройки.
Подключаемся к устройству:
```bash
switch> enable
```
Настраиваем подсистему времени (очень желательно проделать это в первую очередь, хотя бы
для корректной генерации ключей доступа SSH):

!* Устанавливаем текущее время вручную в том случае, если NTP серверы
недоступны
```bash
switch# clock set 16:48:00 23 Feb 2010
switch# configure terminal
```
!* Отключаем переход между зимним и летним временем
```bash
switch(config)# no clock summer-time
```
!* Указываем на наши NTP серверы
```bash
switch(config)# ntp server ip.ntp0.server prefer
switch(config)# ntp server ip.ntp1.server
```
!* Приправляем конфигурацию опциями
```bash
switch(config)# service timestamps debug datetime localtime msec
```
Настраиваем подсистему журналирования событий для отправки сведений на удалённый сервер
Unix Syslog:
```bash
switch(config)# service sequence-numbers
switch(config)# service timestamps log datetime msec localtime showtimezone
switch(config)# logging rate-limit all 10
switch(config)# logging facility local1
switch(config)# logging trap debugging
switch(config)# logging ip.syslog0.server
switch(config)# logging ip.syslog0.server
switch(config)# logging on
```
Настраиваем SNMP для сбора статистики с устройства:
!* Создаем "лист доступа" для SNMP
```bash
switch(config)# access-list 11 permit host ip.snmp0.server
switch(config)# access-list 11 permit host ip.snmp1.server
```
!* Описываем базовые параметры (ключевое слово, уровень доступа, "лист
доступа")
```bash
switch(config)# snmp-server community password ro 11
```
!* Описываем необязательные параметры
```bash
switch(config)# no snmp-server location
switch(config)# no snmp-server contact
```
Устанавливаем имя, доменное имя и параметры разрешения имен:
```bash
switch(config)# hostname sw0
switch(config)# ip domain-name domain.name
switch(config)# ip name-server ip.ns0.server
switch(config)# ip name-server ip.ns1.server
switch(config)# ip domain-lookup
```
Генерируем RSA ключи для подключения к устройству по протоколу SSH и описываем параметры
работы транспорта:
!* Проверяем, нет ли у нас уже ключей
```bash
switch# show crypto key mypubkey rsa
```
!* Удаляем ненужные ключи
```bash
switch(config)# crypto key zeroize rsa
```
!* Генерируем RSA ключ (в процессе он будет именован как:
hostname.domain.name) общий для всех функций устройства
```bash
switch(config)# crypto key generate rsa modulus 1024
```
!* Указываем применять только SSH второй версии
```bash
switch(config)# ip ssh version 2
```
!* Устанавливаем время отключения сессии
```bash
switch(config)# ip ssh time-out 120
```
Устанавливаем параметры аутентификации:
!* Отключаем "продвинутую" модель аутентификации, оставляя, тем самым,
простейший вариант с локальной аутентификацией по файлу паролей (Особое
внимание следует уделить переключению между режимами аутентификации. Если
некорректно переключится в режим "продвинутой" модели аутентификации "AAA",
то запросто можно потерять доступ к устройству даже на консольном порту)
```bash
switch(config)# no aaa new-model
```
!* Активируем шифрование паролей
```bash
switch(config)# service password-encryption
```
!* Активируем доступ уровня "enable" по паролю "cisco"
```bash
switch(config)# enable secret cisco
```
!* Регистрируем пользователя "cisco" c паролем "cisco" c привелегиями,
практически равным суперпользователю
```bash
switch(config)# username cisco privilege 15 password 0 cisco
```
!* Удалить, в случае необходимости, пользователя можно следующим образом
```bash
switch(config)# no username cisco
```
Описываем конфигурацию консольного последовательного терминального порта:
```bash
switch(config)# line console 0
```
!* Устанавливаем время отключения сессии в тридцать минут для неспешной
работы
```bash
switch(config-line)# exec-timeout 30
switch(config-line)# logging synchronous
```
!* Устанавливаем режим аутентификации с локального файла паролей
```bash
switch(config-line)# login local
switch(config-line)# history size 100
switch(config-line)# speed 9600
switch(config-line)# exit
```
Описываем конфигурации виртуальных терминалов:
```bash
switch(config)# line vty 0 15
switch(config-line)# exec-timeout 10
switch(config-line)# logging synchronous
switch(config-line)# login local
```
!* Указываем на применение только одного вида транспортного протокола из
всех возможных
```bash
switch(config-line)# transport input ssh
switch(config-line)# history size 100
switch(config-line)# exit
switch(config)# no ip http server
switch(config)# no ip tftp boot-interface
switch(config)# ip subnet-zero
```
Указываем маршрут "по умолчанию":
```bash
switch(config)# ip default-gateway ip.gateway
switch(config)# exit
switch# copy running-config startup-config
```
