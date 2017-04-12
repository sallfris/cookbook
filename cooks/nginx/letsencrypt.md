Letsencrypt + Nginx proxy
=========================
Установка Certbot
-----------------
Ubuntu:
```
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot 
```

Debian:
```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x certbot-auto
$ mv -f certbot-auto /usr/local/bin/
$ ln -s /usr/local/bin/certbot-auto /usr/local/bin/certbot
```

Настройка Letsencrypt
---------------------
Создаем директорию для проверки доменов, доступную веб серверу
```
$ mkdir /var/www/.well-known/
```
Настраиваем Letsencrypt, правим файл `/etc/letsencrypt/cli.ini`. Если файла нет создаем его.

```
authenticator = webroot
webroot-path = /var/www
post-hook = service nginx reload
text = True
```

Настройка Nginx на получение сертификатов
-----------------------------------------
Создадим файл с настройками для директории `/var/www/.well-known/`
```
$ touch  /etc/nginx/acme
```
Содержимое файла `/etc/nginx/acme`:
```
location /.well-known {
     root /var/www;
 }
```

Конфигурируем хосты на получение сертификата, подключаем файл `acme` ко всем доменам, к которым хотим получить сертификат.

```
server {
        listen 80;
        server_name example.ru;
        client_max_body_size 32m;
        # Подключаем файл с настройками для директории .well-known 
        include acme;
        location / {
                return 301 https://example.ru$request_uri;
        }
}
server {
        listen 443 ssl;
        server_name example.ru;
        client_max_body_size 32m;
        # Полученные сертификаты будут лежать по данному пути в подпапке с названием домена
        #ssl_certificate /etc/letsencrypt/live/example.ru/fullchain.pem;
        #ssl_certificate_key /etc/letsencrypt/live/example.ru/privkey.pem;
        #ssl_trusted_certificate /etc/letsencrypt/live/example.ru/chain.pem;

        ssl_stapling on;
        ssl_stapling_verify on;

        # исключим возврат на http-версию сайта
        add_header Strict-Transport-Security "max-age=31536000";

        # явно "сломаем" все картинки с http://
        add_header Content-Security-Policy "block-all-mixed-content";
        # Подключаем файл с настройками для директории .well-known 
        include acme;
        location / {
                #Перенаправляем остальные запросы на реальный сервер
                proxy_pass http://192.168.101.101;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $remote_addr;
                proxy_connect_timeout 120;
                proxy_send_timeout 120;
                proxy_read_timeout 180;
        }
}

```
Перезапускаем nginx
```
$ sudo service nginx restart
```

Получаем сертификаты
--------------------
Получаем сертификаты для всех необходимых доменов, в первый раз потребуется ввести email для регистрации и согласится с условиями
```
 certbot certonly -d example.ru -d example2.ru
```

Полученые сертификаты лежат в дериктории соответсвующей имени домена в каталоге `/etc/letsencrypt/live/`. Добавим их в конфигурационный файл виртуального хоста
```
        ssl_certificate /etc/letsencrypt/live/example.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.ru/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/example.ru/chain.pem;
```

и перезагрузим конфигурацию _nginx_
```
$ sudo service nginx reload
```

Обновление сертификатов
-----------------------
Все полученные сертификаты можно обновить командой

```
$ sudo certbot renew
```
Проверить работу команды, без реального обновления, можно добавлением ключа `--dry-run`

```
$ sudo certbot renew --dry-run
```

Команда проверяет необходимость обновления сертификата для всех доменов, и в случае необходимости обновляет их.

Для того чтобы обновление происходило автоматически добавим в cron задание 

```
$ sudo crontab -e
```
```
00 1 * * 1 /usr/local/bin/certbot renew >> /var/log/le-renew.log
10 1 * * 1 /usr/bin/service nginx reload 
```
Теперь каждый понедельник в 1:00 будут проверятся и обновлятся сертификаты, а в 1:10 перезапускатся nginx 