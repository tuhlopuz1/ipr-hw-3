### 4 балла - запуск приложения в docker

Написал Dockerfile. При сборке устанавливаются зависимости из requirements.txt и копируется app.py. При запуске контейнера выполняется команда python app.py

Собрал образ ipr и запустил контейнер ipr-counter. Приветствие передал через переменную окружения GREETING из файла .env. порт 8080 пробросил наружу из порта 8000 контейнера. 

docker build -t ipr .
docker run -d --name ipr-counter --env-file .env -p 8080:8000 ipr

Проверил работу приложения запросом curl localhost:8080. в ответе получил ответ 200 с нужным greeting. далее дважды выполнил curl -X POST localhost:8080/increment. Как и ожидалось, первый запрос вернул count: 1, второй - count: 2.

Вывод команды docker logs ipr-counter приведен ниже. 

$ docker logs ipr-counter
2026-11-23 11:19:58,374 listening port=8000 storage=sqlite
2026-11-23 11:20:09,374 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-23 11:20:46,125 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-23 11:20:50,809 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -

### 6 баллов - docker compose и сохранение данных

Настроил запуск приложения через compose.yaml. Приветствие GREETING и внешний порт HOST_PORT задаются в .env. Командой docker-compose config проверил, что compose подставил приветствие, опубликовал порт и подключил том counter-data к /data, где приложение хранит базу sqlite. 

Запустил приложение командой docker-compose up -d. Первый запрос curl localhost:8080 вернул count: 0. Затем увеличил счетчик и проверил значение:

$ curl -X POST localhost:8080/increment
{"greeting": "hello", "count": 1, "hostname": "285004e92251", "storage": "sqlite"}

$ curl localhost:8080
{"greeting": "hello", "count": 1, "hostname": "285004e92251", "storage": "sqlite"}

Удалил контейнер командой docker-compose down, сохранив том, и снова запустил приложение:

$ docker-compose up -d
[+] Running 2/2
 ✔ Network ipr-hw-3_default  Created                                                                                                   0.1s 
 ✔ Container ipr-hw-3-app-1  Started                                                                                                   0.3s 

$ curl localhost:8080
{"greeting": "hello", "count": 1, "hostname": "ef8a411b6cb1", "storage": "sqlite"}

После создания контейнера hostname изменился, а значение счетчика сохранилось в томе и осталось равно 1.

### 8 баллов - Redis и восстановление после сбоя

Добавил в compose.yaml сервис redis. Приложение подключается к нему по адресу redis://redis:6379/0. Redis записывает данные на диск в режиме AOF и хранит их в томе redis-data. Наружу опубликован только порт приложения 8080.

Запустил сервисы командой docker-compose up -d --wait --build. Команда docker-compose ps показала, что оба сервиса healthy. Первый запрос curl localhost:8080 вернул storage: redis и count: 0. Три вызова curl -X POST localhost:8080/increment последовательно вернули 1 2 3.

Затем пересоздал контейнеры, сохранив том:

$ docker-compose down
$ docker-compose up -d --wait
$ curl localhost:8080
{"greeting": "hello", "count": 3, "hostname": "de2ba4639f75", "storage": "redis"}

Имя контейнера в поле hostname изменилось, а счетчик сохранил значение 3.

Далее остановил redis: docker-compose stop redis и проверил 3 запроса:

$ curl localhost:8080/healthz
{"status": "alive"} - код ответа 200

$ curl localhost:8080/readyz
{"error": "storage_unavailable"} - код ответа 503

$ curl -X POST localhost:8080/increment
{"error": "storage_unavailable"} - код ответа 503

(коды ответов взял из логов контейнера)

Затем запустил только redis командой docker-compose start redis. Когда docker-compose ps снова показало healthy на обоих сервисах, проверил чтение и увеличение счетчика:

$ curl localhost:8080
{"greeting": "hello", "count": 3, "hostname": "de2ba4639f75", "storage": "redis"}

$ curl -X POST localhost:8080/increment
{"greeting": "hello", "count": 4, "hostname": "de2ba4639f75", "storage": "redis"}

Приложение восстановило соединение с Redis без ручного перезапуска. Запрос во время сбоя оставил счетчик равным 3, а после increment он стал 4.