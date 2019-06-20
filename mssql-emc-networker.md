MSSQL backup & restore
======================

Dell EMC Networker
------------------

Программы командной строки

* **nsrsqlsv** - создание резервной копии баз данных
* **nsrsqlrc** - восстановление резервной копии баз данных
* **nsrinfo** - Просмотр содержимого backup-а ([man](http://www.ipnom.com/Legato-NetWorker-Commands/nsrinfo.html), [sample](http://nsrd.info/blog/2009/01/27/basics-listing-files-in-a-backup/))

Установка EMC Networker
-----------------------

### Компоненты установки

Предварительно

* необходимо установить сертификат Verisign для установки Networker
* необходимо установить Net Framework 4 Full для модуля NMM (Networker MSSQL backup)
* отключить UAC

Необходимо установить компоненты EMC (Для ОС начаниая с Windows 2008+)

* Networker клиент соответствующий ОС (пример `дистрибутив\nw9212_win_x64\win_x64\networkr\lgtoclnt-9.2.1.2.exe`)
* Расширенный клиент Networker (пример `дистрибутив\nw9212_win_x64\win_x64\networkr\lgtoxtdclnt-9.2.1.2.exe`)
* Модуль Networker для MSSQL (NMM) (пример `дистрибутив\\nmm9211_win_x64\win_x64\networkr\NWVSS.exe`)

### Установка EMC Networker клиента

* Все шаги по умолчанию
* В последнем диалоге Complete the Setup (см. Networker server list)
	1. Нажать кнопку Select Backup Server ...
	2. В диалоге NetWorker Server Selection добавить сервер _backup-server.com_ в список Selected Servers

### Установка Расширенного клиента Networker

Все шаги по умолчанию

### Установка Модуля Networker для MSSQL (NMM)

Все шаги по умолчанию

В диалоге NMM Installation Options указать

* SQL SSMS Plugin - ok
* Enable Backup Tab for Ad-hoc Backup - ok
* Enable Script at Backup Window - ok

### Добавление доменного пользователя `SQL_NETWORKER`

Для работы необходимо создать/добавить Login в MSSQL, для пользователя DOMAIN\sql_networker с правами sysdba

```sql
USE [master]
GO
CREATE LOGIN [URAL\svc_networker] FROM WINDOWS WITH DEFAULT_DATABASE=[master]
GO
EXEC master..sp_addsrvrolemember @loginame = N'URAL\svc_networker', @rolename = N'sysadmin'
GO
```

### Networker server list

Последним этапом инсталяции Networker клиент просит указать NetWorker сервер, данные изменения сохраняются в файле: `C:\Program Files\EMC NetWorker\nsr\res\servers`

Под Linux - `/nsr/res/servers` по умолчанию данный файл отсутствует

данный файл влияет на настройки безопастности клиента - принимает задания на РК только с указанных серверов (whitelist), если файл пуст или отсутствует, то разрешено всё всем (при условии resolve DNS IP -> Hostname)

### Dell EMC Networker для старых ОС (Windows 2003 / MSSQL 2000+)

Для версии Windows 2003 необходимо использовать старый клиент Networker 8.1

#### mssql batch

Создать файл `nsrsqlsv.bat` в каталоге `C:\Program Files\EMC NetWorker\nsr\bin`

```bat
@ECHO OFF
SETLOCAL

SHIFT
SET arg=%0

:loop
SHIFT

:: end arguments
IF %0.==. GOTO save

:: drop arg -a and corresponding value
if %0 ==-a (
 SHIFT
 GOTO loop
)

:: drop arg -o and corresponding value
if %0 ==-o (
 SHIFT
 GOTO loop
)

SET arg=%arg% %0
GOTO loop

:save
::echo catch args >>C:\temp\nsrargs.txt
::echo %arg% >>C:\temp\nsrargs.txt

"C:\Program Files\EMC NetWorker\nsr\bin\nsrsqlsv.exe" %arg%
ENDLOCAL
```

Во время конфигурации задания РК в консоли networker сервера вместо команды nsrsqlsv.exe использовать nsrsqlsv.bat

Примечания

* В консоли networker-а wizzard баз СУБД не видит, Создать клиента в ручном режиме.
* **Внимание LOG GAP DETECTION не работает**
* Для РК Логов MSSQL использовать sheduler тип: Incr (В отличии от LOG ONLY, LOG ONLY - не использовать)

backup with Networker 8.x / NMM 3
---------------------------------

Пример

<pre style="font-size:80%">
Administrator@<b>HOSTNAME</b> C:\
> nsrsqlsv.exe -l <b>diff</b> -s "<b>backup-server.com</b>" -c "<b>backup-client.com</b>"  "MSSQL:<b>DBName</b>"
43708:(pid 57140):Start time: Thu Jul 05 13:00:29 2018
43621:(pid 57140):Computer Name: <b>HOSTNAME</b>     User Name: Administrator
87749:(pid 57140):Detected application flag NSR_SKIP_SIMPLE_DB set to TRUE in NW resources for client <b>hostname</b>.
            NSR_BACKUP_LEVEL: 1;
                  NSR_CLIENT: <b>backup-client.com</b>;
           NSR_DIRECT_ACCESS: default;
            NSR_SAVESET_NAME: "MSSQL:<b>DBName</b>";
                  NSR_SERVER: <b>backup-server.com</b>;
37994:(pid 57140):Backing up Energy...
4690:(pid 57140):BACKUP database [<b>DBName</b>] TO virtual_device='Legato#8c878897-b487-41ac-855b-119c1cc32996' WITH name=N'LegatoNWMS
QL', description=N'MSSQL:<b>DBName</b>' ,differential
53085:(pid 57140):Backing up of Energy succeeded.
nsrsqlsv: MSSQL:Energy level=1, 1985 MB 00:04:10      1 file(s)
43709:(pid 57140):Stop time: Thu Jul 05 13:04:40 2018
</pre>

Параметры командной строки

* **-l _level_** - где _level_ - тип резервной копии:
	* **full** - полное резервная копия
	* **diff** - diff копия базы
	* **incr** - копия лога транзакций
* **-s _backup_server_** - указывает Networker сервер
* **-c _backup_client_** - указывает клиента (dns имя) клиента Networker сервера, т.е. имя MSSQL сервера
* **"MSSQL:_DBName_"** - указывает имя базы данных. Варианты:
	* MSSQL$**_Instance_**:DBName - Случай с именнованым экземпляром
	* **"MSSQL:"** - Создать копии всех баз
* **-k** - проверять checksum при создании РК
* **-u** - продолжать backup при обнаружении ошибок (checksum)
* **-h _dbname_** - исключить из копий указанную базу: `nsrsqlsv -s bv-customer.belred.emc.com -h master -h model MSSQL:`
* **-v** - отображть более подробную информацию
* **-U _sql_login_ -P _sql_pswd_** - Использовать login/password при подключении к MSSQL серверу
* **-R** - не усекать лог транзакций (`NO_TRUNCATE`)
* **-y Ndays** - указать сколько дней хранить, где **N** - кол-во дней, пр: `-y 2days`
* **-b _nwsn1-aftd-data_** - указывает в какой пул записать данные, _nwsn1-aftd-data_ - имя пула

restore with Networker 8.X / NMM 3
----------------------------------

### Просмотр резервных копий

<pre style="font-size:80%">
Administrator@HOSTNAME C:\
> mminfo -s <b>backup-server.com</b> -c <b>backup-client.com</b> -r "name,ssid,level,fragsize,savetime(25),nsavetime" -q "name=MSSQL:<b>DBName</b>" -o tR

 name                          ssid         lvl   size      date     time        save time
MSSQL:<b>DBName</b>                   1548044976  incr   2 KB     2018-07-11 05:09:20a 1531267760
MSSQL:<b>DBName</b>                   1917143411     1 2943 MB    2018-07-11 05:04:03a <b>1531267443</b>
MSSQL:<b>DBName</b>                   893705780   incr   2 KB     2018-07-10 09:26:28p 1531239988
MSSQL:<b>DBName</b>                   2336545205     1 2045 MB    2018-07-10 09:07:17p 1531238837
MSSQL:<b>DBName</b>                   1832883200  full  10 GB     2018-07-06 09:08:32p <b>1530893312</b>
MSSQL:<b>DBName</b>                   1832883200  full 8999 MB    2018-07-06 09:08:32p 1530893312
</pre>

### Востановление базы из FULL с опцией NORECOVERY

<pre style="font-size:80%">
Administrator@HOSTNAME D:\soft\procexp
> nsrsqlrc -z <b>-t 1530893312</b> -S <b>norecovery</b> -s "<b>backup-server.com</b>" -c "<b>backup-client.com</b>" -C "LOGIC_DATA_FILE1=E:\Databases\data01.mdf,LOGIC_LOG_FILE1=F:\Databases\log01.ldf" -d "MSSQL:<b>DBNameFrom</b>" "MSSQL:<b>DBNameTo</b>"
43708:(pid 3544):Start time: Thu Jul 05 12:54:25 2018
43621:(pid 3544):Computer Name: HOSTNAME     User Name: Administrator
                  NSR_CLIENT: <i>backup-client.com</i>;
                  NSR_SERVER: <i>backup-server.com</i>;
37725:(pid 3544):Recovering database '<i>DBNameFrom</i>' into '<i>DBNameTo</i>' ...
4690:(pid 3544):RESTORE database [<i>DBNameTo</i>] FROM virtual_device='Legato#f9dd5ce1-24c1-41a6-b737-f8ee0657fd4e'  WITH move '<i>LOGIC_DATA_FILE1</i>' to '<i>E:\Databases\data01.mdf</i>', move '<i>LOGIC_LOG_FILE1</i>' to '<i>F:\Databases\log01.ldf</i>', <i>norecovery</i>
35866:(pid 3544):The restore of database '<i>DBNameTo</i>' completed successfully.
29309:(pid 3544):
Received  1968 KB  1 item(s)  from NSR server.
43709:(pid 3544):Stop time: Thu Jul 05 12:54:40 2018
</pre>

### Востановление базы из DIFF

<pre style="font-size:80%">
Administrator@<b>HOSTNAME</b> C:\
> nsrsqlrc -z <b>-t 1531267443</b> -s "<b>backup-server.com</b>" -c "<b>backup-client.com</b>" "MSSQL:<b>DBName</b>"
43708:(pid 5024):Start time: Thu Jul 05 13:07:58 2018
43621:(pid 5024):Computer Name: HOSTNAME     User Name: Administrator
                  NSR_CLIENT: <i>backup-client.com</i>;
                  NSR_SERVER: <i>backup-server.com</i>;
37721:(pid 5024):Recovering database '<i>DBName</i>' ...
65175:(pid 5024):RESTORE database [<i>DBName</i>] from virtual_device='Legato#a75318f2-27fd-41ff-add6-10d0acb8df5d' (differential)
35866:(pid 5024):The restore of database '<i>DBName</i>' completed successfully.
29309:(pid 5024):
Received  1985 MB  1 item(s)  from NSR server.
43709:(pid 5024):Stop time: Thu Jul 05 13:09:49 2018
</pre>

Просмотр содержимого backup-а (Networker 9+)
--------------------------------------------

<pre style="font-size:80%">
nsrinfo -vV -s <b>backup-server.com</b> -n all -t 1528329669 <b>backup-client.com</b>
</pre>

Параметры

* -s backup-server.com - адрес networker сервера
* -t 1528329669 - значение колонки save time из результата команды `mminfo .. -r "..nsavetime"`
* backup-client.com - имя клиента networker

Вывод команды

<pre style="font-size:80%">
scanning client `<b>backup-client.com</b>' for savetime 1528329669(07.06.2018 5:01:09) on server <b>backup-server.com</b>
XBSA file `MSSQL:/msdb', size=271896844, off=0, app=mssql(14), copyID = 1528329669.1528329670
XBSA file `MSSQL:/msdb%/files.1528329669.1528329670', size=452, off=271896844, app=mssql(14), copyID = 1528329669.1528329671, logical file list:
 name='MSDBData', filename='E:\MSSQL\MSSQL13.MSSQLSERVER\MSSQL\DATA\MSDBData.mdf', filegroup='PRIMARY', size=1852440576
 name='MSDBLog', filename='E:\MSSQL\MSSQL13.MSSQLSERVER\MSSQL\DATA\MSDBLog.ldf', size=20578304
</pre>

В данном примере

* `MSSQL:/msdb` - имя базы
    * `size` - размер backup файла MSSQL
* `MSSQL:/msdb%/files.1528329669.1528329670` - информация по файлам внутри backup файла MSSQL
    * `name='MSDBData'` - логическое имя MSSQL файла базы данных msdb
    * `filename='E:\MSSQL\MSSQL13.MSSQLSERVER\MSSQL\DATA\MSDBData.mdf'` - физическое расположение файла данных базы msdb
    * `filegroup='PRIMARY'` - имя файловой группы базы msdb
    * `size=1852440576` - размер файла

MSSQL flat file recovery
------------------------

Восстановление резервной копии как самостоятельного файла р.копии MSSQL

<pre style="font-size:80%">
Administrator@HOSTNAME C:\

$ nsrsqlrc.exe  -s <b>backup-server.com</b> -c <b>backup-client.com</b> -t "06/06/2018 05:02:04 PM" -S normal -a "SKIP_CLIENT_RESOLUTION=TRUE" -a <b>"FLAT_FILE_RECOVERY=TRUE"</b> -a <b>FLAT_FILE_RECOVERY_DIR=F:\restore</b> MSSQL:<b>DBName</b>

Check the detailed logs that are located at 'C:\Program Files\EMC NetWorker\nsr\applogs\nsrsqlrc.log'.

The 'Skip client resolution' option is set to true. Client name '<i>backup-client.com</i>' will be used for recovery.

Start time: Thu Jun 07 10:47:16 2018

Computer name: HOSTNAME     User name: Administrator

Version information for C:\Program Files\EMC NetWorker\nsr\bin\nsrsqlrc.exe:    Original file name: nsrsqlrc.exe        Version: 9.2.1 (ntx64)  Comments: Supporting Microsoft Volume Shadow Copy Service

52701:(pid 10832): Command line:
  nsrsqlrc.exe -s <i>backup-server.com</i> -c <i>backup-client.com</i> -t 06/06/2018 05:02:04 PM -S normal -a SKIP_CLIENT_RESOLUTION=TRUE -a FLAT_FILE_RECOVERY=TRUE -a
 FLAT_FILE_RECOVERY_DIR=<i>F:\restore</i> MSSQL:<i>DBName</i>
NSR_SERVER : <i>backup-server.com</i>
.
Recovering database '<i>DBName</i>' ...

Flat file recovery will be performed to 'F:\restore' directory.

151552:(pid 10832): File '1528286524_DBName_full.bak' in the directory 'F:\restore\<i>backup-client.com</i>\DEFAULT_INSTANCE' has been opened for writing the 
backup '/<i>DBName</i>'.

148972:(pid 10832): 386935808 bytes were written to file '1528286524_DBName_full.bak' in the directory 'F:\restore\<i>backup-client.com</i>\DEFAULT_INSTANCE'.


The restore of database '<i>DBName</i>' completed successfully.

Stop time: Thu Jun 07 10:47:46 2018
</pre>
