duplicitybkp
=========

Backup for duplicity

Requirements
------------


Role Variables
--------------


Dependencies
------------


Example Playbook
----------------

У роли есть несколько "внутренних роли" определяемые переменной bkptype

bkptype:

 - mysql - выкладывает скрип для бэкапа mysql субд. скрипту передаются реквизиты доступа к `mysql: host,port,user,password` скрипт считывает дополнительный файл из `/etc/{{ bkpprefix }}mysqlbkp/mysql`.
    - Так же устанавливается крон задание для запуска скрипта.
 
 - mysqlclient - заполняет файл /etc/{{ bkpprefix }}mysqlbkp/mysql переданными реквизитами
 

License
-------

GPLv3

Author Information
------------------

xoy dot mail at gmail dot com
