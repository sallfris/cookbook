Примеры конфигурационных файлов nginx
======================================

Yii2
----

Nginx SSL
---------
```
# read more at https://terrty.net/2014/ssl-tls-in-nginx/
# latest version on https://gist.github.com/paskal/628882bee1948ef126dd/126e4d1daeb5244aacbbd847c5247c2e293f6adf
# security test score: https://www.ssllabs.com/ssltest/analyze.html?d=[SITE]
# your nginx version might not have all directives included, test this configuration before using in production against your nginx:
# $ nginx -c /etc/nginx/nginx.conf -t

server {
        # public key, contains your public key and class 1 certificate, to create:
        # (example for startssl)
        # $ (cat example.com.pem & wget -O - https://www.startssl.com/certs/class1/sha2/pem/sub.class1.server.sha2.ca.pem) | tee -a /etc/nginx/ssl/domain.pem > /dev/null
        ssl_certificate /etc/nginx/ssl/domain.pem;

        # private key (decoded), decode encoded with RSA key with command:
        # $ openssl rsa -in decoded.key -out domain.key
        ssl_certificate_key /etc/nginx/ssl/domain.key;

        # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
        # to generate your dhparam.pem file, run in the terminal:
        # $ openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;

        # don't forget to set secure rights to these files:
        # $ chmod 400 /etc/nginx/ssl/*

        # http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache
        # make it bigger for more sessions, one megabyte for ~ 4000 session
        ssl_session_cache shared:SSL:100m;
        ssl_session_timeout 60m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

        # ciphers are latest modern from https://wiki.mozilla.org/Security/Server_Side_TLS (only place you can trust on web)
        # working example:
        # ssl_ciphers 'EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DES';       
        ssl_ciphers '<paste intermediate ciphersuite here>';
        ssl_prefer_server_ciphers on;

        # OCSP Stapling ---
        # fetch OCSP records from URL in ssl_certificate and cache them
        ssl_stapling on;
        ssl_stapling_verify on;
        # dns resolver, we're using Google IPv4 and IPv6 servers
        resolver 8.8.8.8 [2001:4860:4860::8888];
        # verify chain of trust of OCSP response using Root CA and Intermediate certs, example for StartSSL:
        # $ wget -O - https://www.startssl.com/certs/ca.pem https://www.startssl.com/certs/class1/sha2/pem/sub.class1.server.sha2.ca.pem | tee -a ca-certs.pem > /dev/null
        ssl_trusted_certificate /etc/nginx/ssl/root_CA_cert_plus_intermediates;

        # consider turning 'deferred' off on old versions of nginx if you occur any problems
        listen 443 deferred spdy ssl;
        listen [::]:443 deferred spdy ssl ipv6only=on;
        server_name example.com;
        root /var/local/www/example;
        index index.html;
        autoindex off;
        charset utf-8;

        #don't send the nginx version number in error pages and Server header
        server_tokens off;

        # https://www.owasp.org/index.php/List_of_useful_HTTP_headers

        # enabling HSTS(HTTP Strict Transport Security)
        # https://developer.mozilla.org/en-US/docs/Web/Security/HTTP_strict_transport_security
        add_header Strict-Transport-Security 'max-age=31536000' always;
        
        
        # enabling Public Key Pinning Extension for HTTP (HPKP)
        # https://developer.mozilla.org/en-US/docs/Web/Security/Public_Key_Pinning
        # tool for checking and generating proper certificates: https://report-uri.io/home/tools
        # to generate use on of these:
        # $ openssl rsa  -in my-website.key -outform der -pubout | openssl dgst -sha256 -binary | base64
        # $ openssl req  -in my-website.csr -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64
        # $ openssl x509 -in my-website.crt -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64
        add_header Public-Key-Pins 'pin-sha256="base64+info1="; max-age=31536000' always;
        
        
        # config to don't allow the browser to render the page inside an frame or
        # iframe and avoid clickjacking http://en.wikipedia.org/wiki/Clickjacking
        # if you need to allow [i]frames, you can use SAMEORIGIN
        # or set an uri with ALLOW-FROM uri
        # warning, this option breaking some analitics tools
        add_header X-Frame-Options DENY;

        # when serving user-supplied content, include a
        # X-Content-Type-Options: nosniff header along with the Content-Type:
        # header to disable content-type sniffing on some browsers.
        # https://github.com/blog/1482-heads-up-nosniff-header-support-coming-to-chrome-and-firefox
        add_header X-Content-Type-Options nosniff;

        # this header enables the Cross-site scripting (XSS) filter, it's usually
        # enabled by default anyway, so the role of this header is to re-enable
        # the filter for this particular website if it was disabled by the user.
        add_header X-XSS-Protection "1; mode=block";

        location / {
                # try_files might be dangerous, please read: http://blog.volema.com/nginx-insecurities.html
                try_files $uri $uri/ =404;
        }

        # deny access to files, starting with dot (hidden) or ending with ~ (temp)

        location ~ /\. {
                access_log off;
                log_not_found off;
                deny all;
        }

        location ~ ~$ {
                access_log off;
                log_not_found off;
                deny all;
        }

        # block of rules for static content

        location ~ /{favicon.ico|favicon.png|robots.txt}$ {
                access_log off;
                log_not_found off;
                expires 1y;
                add_header Cache-Control public,max-age=259200;
        }

        location ~*  \.(jpg|jpeg|png|gif|ico|css|js|mp3)$ {
                expires 30d;
                add_header Cache-Control public,max-age=259200;
        }

}

server {
        # catch all unsecure requests (both IPv4 and IPv6)
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        # this means example.com, *.example.com
        server_name .example.com;

        # permanently redirect client to https version of the site
        return 301 https://example.com;
}
```

Nginx Proxy
-----------
```
server {
        listen 80;
        server_name sys.foodison.ru;
        client_max_body_size 32m;
        include acme;
        location / {
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                proxy_connect_timeout 120;
                proxy_send_timeout 120;
                proxy_read_timeout 180;
        }
}

```
