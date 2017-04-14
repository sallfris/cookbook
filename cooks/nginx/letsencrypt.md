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
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name example.ru;
        client_max_body_size 32m;
        include acme;
        location / {
                return 301 https://example.ru$request_uri;
        }
}
server {
        listen 443 ssl;
        listen [::]:443 deferred spdy ssl ipv6only=on;
        server_name example.ru;
        client_max_body_size 32m;
        server_tokens off;
        
        add_header Strict-Transport-Security 'max-age=31536000';
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Content-Security-Policy "block-all-mixed-content";

        ssl_certificate /etc/letsencrypt/live/example.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.ru/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/example.ru/chain.pem;

        # Для установки единого хранилища ссесий 
        # ssl_session_cache shared:SSL:100m;
        
        ssl_session_timeout 60m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        
        //Берем нужную строку отсюда  https://wiki.mozilla.org/Security/Server_Side_TLS#Recommended_configurations
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
        ssl_prefer_server_ciphers on;

        ssl_stapling on;
        ssl_stapling_verify on;

        resolver 8.8.8.8 [2001:4860:4860::8888];


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

Для того чтобы обновление происходило автоматически, добавим в cron задание 

```
$ sudo crontab -e
```
```
00 1 * * 1 /usr/local/bin/certbot renew >> /var/log/le-renew.log
10 1 * * 1 /usr/bin/service nginx reload 
```
Теперь каждый понедельник в 1:00 будут проверятся и обновлятся сертификаты, а в 1:10 перезапускатся nginx 