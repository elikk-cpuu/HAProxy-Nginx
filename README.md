# Практическая работа по HAProxy и Nginx

Выполнил: **Нестеренко Андрей**

---

## Цель работы

В ходе практической работы были выполнены следующие задачи:

* запущены simple Python server на разных портах;
* установлен и настроен HAProxy;
* настроена балансировка Round-robin на 4 уровне;
* настроена балансировка Weighted Round Robin на 7 уровне;
* настроена балансировка HTTP-трафика только для домена `example.local`;
* настроена связка HAProxy + Nginx;
* настроена выдача `.jpg` файлов средствами Nginx;
* настроена переадресация остальных запросов с Nginx на HAProxy;
* настроена маршрутизация запросов в HAProxy по доменам `example1.local` и `example2.local`.

---

## Задание 1

Были запущены два simple Python server на одной виртуальной машине на разных портах:

| Сервер   | Адрес     | Порт |
| -------- | --------- | ---- |
| Server 1 | 127.0.0.1 | 8888 |
| Server 2 | 127.0.0.1 | 9999 |

Для первого сервера был создан файл `index.html` со следующим содержимым:

```text
Server 1 Port 8888
```

Для второго сервера был создан файл `index.html` со следующим содержимым:

```text
Server 2 Port 9999
```

Серверы были запущены командами:

```bash
cd ~/server1
python3 -m http.server 8888
```

```bash
cd ~/server2
python3 -m http.server 9999
```

Была настроена балансировка Round-robin на 4 уровне через HAProxy.

Конфигурационный файл HAProxy:

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend task1_frontend
    bind *:8080
    mode tcp
    default_backend task1_backend

backend task1_backend
    mode tcp
    balance roundrobin
    server s1 127.0.0.1:8888 check
    server s2 127.0.0.1:9999 check
```

Проверка конфигурации HAProxy:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

При обращении к HAProxy на порт `8080` запросы перенаправлялись на разные Python-серверы.

Проверка:

```bash
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
```

Результат:

```text
Server 1 Port 8888
Server 2 Port 9999
Server 1 Port 8888
Server 2 Port 9999
```

Скриншот проверки Round-robin балансировки:

<img src = "img/Pasted image 20260609002946.png" width = 50%>

---

## Задание 2

Были запущены три simple Python server на одной виртуальной машине на разных портах:

| Сервер   | Адрес     | Порт | Вес |
| -------- | --------- | ---- | --- |
| Server 1 | 127.0.0.1 | 8888 | 2   |
| Server 2 | 127.0.0.1 | 9999 | 3   |
| Server 3 | 127.0.0.1 | 7777 | 4   |

Для третьего сервера был создан файл `index.html` со следующим содержимым:

```text
Server 3 Port 7777
```

Третий сервер был запущен командой:

```bash
cd ~/server3
python3 -m http.server 7777
```

В файл `/etc/hosts` была добавлена запись для домена `example.local`:

```text
127.0.0.1 example.local
```

Была настроена балансировка Weighted Round Robin на 7 уровне. HAProxy балансирует только HTTP-трафик, адресованный домену `example.local`.

Конфигурационный файл HAProxy:

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend task2_frontend
    bind *:8080
    mode http

    acl is_example_local hdr(host) -i example.local:8080 example.local
    use_backend task2_backend if is_example_local

    default_backend deny_backend

backend task2_backend
    mode http
    balance roundrobin
    server s1 127.0.0.1:8888 weight 2 check
    server s2 127.0.0.1:9999 weight 3 check
    server s3 127.0.0.1:7777 weight 4 check

backend deny_backend
    mode http
    http-request deny
```

Проверка конфигурации HAProxy:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

Проверка обращения с использованием домена `example.local`:

```bash
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
curl http://example.local:8080
```

В результате запросы распределялись между тремя серверами с весами 2, 3 и 4.

Скриншот проверки Weighted Round Robin для `example.local`:

<img src = "img/Pasted image 20260609002946.png" width = 50%>

Проверка обращения без домена `example.local`:

```bash
curl http://127.0.0.1:8080
```

Результат:

```text
403 Forbidden
```

Скриншот проверки запрета обращения без домена `example.local`:

Скриншот проверки Weighted Round Robin для `example.local`:

<img src = "img/Pasted image 20260609004133.png" width = 50%>

---

## Задание 3*

Была настроена связка HAProxy + Nginx.

Схема работы:

```text
Клиент → Nginx :80
              ├── .jpg файлы отдаются самим Nginx из /var/www
              └── остальные запросы перенаправляются на HAProxy :8080
                                                └── Python servers 8888 / 9999
```

В директорию `/var/www/` была добавлена тестовая картинка:

```bash
sudo mkdir -p /var/www
sudo tee /var/www/test.jpg.base64 > /dev/null
sudo base64 -d /var/www/test.jpg.base64 | sudo tee /var/www/test.jpg > /dev/null
sudo rm /var/www/test.jpg.base64
```

Конфигурационный файл HAProxy:

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend nginx_to_haproxy
    bind 127.0.0.1:8080
    mode http
    default_backend python_servers

backend python_servers
    mode http
    balance roundrobin
    server s1 127.0.0.1:8888 check
    server s2 127.0.0.1:9999 check
```

Конфигурационный файл Nginx:

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /var/www;

    location ~* \.jpg$ {
        try_files $uri =404;
    }

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Проверка конфигурации:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy

sudo nginx -t
sudo systemctl restart nginx
```

Проверка выдачи `.jpg` файла самим Nginx:

```bash
curl -I http://127.0.0.1/test.jpg
```

Результат:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: image/jpeg
```

Скриншот проверки выдачи `.jpg` файла:

<img src = "img/Pasted image 20260609010531.png" width = 50%>

Проверка переадресации остальных запросов с Nginx на HAProxy и далее на Python-серверы:

```bash
curl http://127.0.0.1/
curl http://127.0.0.1/
curl http://127.0.0.1/
curl http://127.0.0.1/
```

Результат:

```text
Server 1 Port 8888
Server 2 Port 9999
Server 1 Port 8888
Server 2 Port 9999
```

Скриншот проверки переадресации запросов на Python-серверы:

<img src = "img/Pasted image 20260609010623.png" width = 50%>

---

## Задание 4*

Были запущены четыре simple Python server на разных портах.

Первые два сервера выдают страницу сайта `example1.local`.

| Сервер      | Адрес     | Порт | Содержимое index.html   |
| ----------- | --------- | ---- | ----------------------- |
| example1_s1 | 127.0.0.1 | 8001 | example1.local server 1 |
| example1_s2 | 127.0.0.1 | 8002 | example1.local server 2 |

Вторые два сервера выдают страницу сайта `example2.local`.

| Сервер      | Адрес     | Порт | Содержимое index.html   |
| ----------- | --------- | ---- | ----------------------- |
| example2_s1 | 127.0.0.1 | 8003 | example2.local server 1 |
| example2_s2 | 127.0.0.1 | 8004 | example2.local server 2 |

Серверы были запущены командами:

```bash
cd ~/example1_s1
python3 -m http.server 8001
```

```bash
cd ~/example1_s2
python3 -m http.server 8002
```

```bash
cd ~/example2_s1
python3 -m http.server 8003
```

```bash
cd ~/example2_s2
python3 -m http.server 8004
```

В файл `/etc/hosts` были добавлены записи:

```text
127.0.0.1 example1.local
127.0.0.1 example2.local
```

В HAProxy были настроены два backend:

* `example1_backend` — для сайта `example1.local`;
* `example2_backend` — для сайта `example2.local`.

Frontend HAProxy определяет нужный backend по HTTP-заголовку `Host`.

Конфигурационный файл HAProxy:

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode http
    option httplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend sites_frontend
    bind *:8080
    mode http

    acl host_example1 hdr(host) -i example1.local:8080 example1.local
    acl host_example2 hdr(host) -i example2.local:8080 example2.local

    use_backend example1_backend if host_example1
    use_backend example2_backend if host_example2

    default_backend deny_backend

backend example1_backend
    mode http
    balance roundrobin
    server example1_s1 127.0.0.1:8001 check
    server example1_s2 127.0.0.1:8002 check

backend example2_backend
    mode http
    balance roundrobin
    server example2_s1 127.0.0.1:8003 check
    server example2_s2 127.0.0.1:8004 check

backend deny_backend
    mode http
    http-request deny
```

Проверка конфигурации HAProxy:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

Проверка сайта `example1.local`:

```bash
curl http://example1.local:8080
curl http://example1.local:8080
curl http://example1.local:8080
curl http://example1.local:8080
```

Результат:

```text
example1.local server 1
example1.local server 2
example1.local server 1
example1.local server 2
```

Скриншот проверки сайта `example1.local`:

<img src = "img/Pasted image 20260609011047.png" width = 50%>

Проверка сайта `example2.local`:

```bash
curl http://example2.local:8080
curl http://example2.local:8080
curl http://example2.local:8080
curl http://example2.local:8080
```

Результат:

```text
example2.local server 1
example2.local server 2
example2.local server 1
example2.local server 2
```

Скриншот проверки сайта `example2.local`:

<img src = "img/Pasted image 20260609011101.png" width = 50%>
Проверка обращения без нужного домена:

```bash
curl http://127.0.0.1:8080
```

Результат:

```text
403 Forbidden
```

Скриншот проверки обращения без домена:

<img src = "img/Pasted image 20260609011116.png" width = 50%>

---

## Итог

В результате выполнения практической работы была настроена балансировка нагрузки с помощью HAProxy на 4 и 7 уровнях. Также была настроена связка Nginx + HAProxy, где Nginx отдаёт статические `.jpg` файлы, а остальные запросы передаёт на HAProxy. Дополнительно была настроена маршрутизация запросов в HAProxy по доменным именам `example1.local` и `example2.local`.
