Установка сервера Linux, nginx, MySQL, PHP-FPM
======================================
Установка nginx
---------------
Обновляем список пакетов:

_Ubuntu:_
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

Устанавливаем nginx:

_Ubuntu:_
```
sudo apt-get install nginx nginx-extras
```

Если стоял Apache, останавливаем apache и убираем из автозагрузки:

_Ubuntu 14.04:_
```
sudo service apache2 stop
sudo update-rc.d -f apache2 remove
```

Стартуем nginx

_Ubuntu:_
```
# service nginx start
# service nginx status
 * nginx is running

```

Установка mysql
---------------

При установке пароль можно не вводить, его можно задать при запуске `mysql_secure_installation`. 

_Ubuntu:_
```
sudo apt-get install mysql-server
sudo mysql_install_db
sudo mysql_secure_installation
```

Установка PHP-FPM
-----------------

_Ubuntu:_
```
sudo apt-get install php5-cli php5-common php5-mysql php5-gd php5-fpm php5-cgi php5-fpm php-pear php5-mcrypt
```

Настройка PHP-FPM
-----------------

Редактируем файл `/etc/php5/fpm/php.ini`
```
cgi.fix_pathinfo = 0
post_max_size = 64M
upload_max_filesize = 64M
```

Редактируем `/etc/php5/fpm/pool.d/www.conf`
```
security.limit_extensions = .php .php3 .php4 .php5
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Перезапускаем 
```
sudo service php5-fpm restart
```