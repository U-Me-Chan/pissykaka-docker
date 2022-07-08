Бекенд pissykaka может быть найден
[здесь](https://github.com/U-Me-Chan/pissykaka).

Спасибо @ridouchire за то, что подготовил конфиги, докер-композы и хотя бы один
раз показал мне как это делается.

# Инструкция по запуску бекенда pissykaka

Возможно людям, ежедневно сталкивающимся с разработкой на PHP это всё будет
очевидно, но тем не менее, этот репозиторий существует, чтобы помочь хотя бы
тем, кто вообще ничего не понимает, то есть мне самому.

Здесь описано разворачивание приложения с логинами и паролями типа root:root,
так что это подойдёт только для разработки, то есть локального тестирования.
Для продуктового развёртывания лучше разобраться что здесь происходит и
задействовать нетривиальные логины и пароли.

## Инструменты

В системе должен быть установлен `docker` и `docker-compose`. Распинаться тут
про это не буду - почитайте сами в интернете.

## 0 Клонирование репозитория

Вся работа начинается с клонирования данного репозитория и перехода в его
директорию.

    git clone https://github.com/U-Me-Chan/pissykaka-docker
    cd pissykaka-docker/

## 1 Подготовка исходников в директории `./public`

В данном пунтке подготавливается директория `./public`, в которой будут
расположены исходники. Можно склонировать репозиторий
`https://github.com/ridouchire/pissykaka.git` прямо в `public`, а можно сделать
символьную ссылку на внешнюю директорию с исходниками вместо клонирования.

### 1.1 Вариант 1: клонирование в `./public`:

    git clone https://github.com/U-Me-Chan/pissykaka.git pubilc

### 1.2 Вариант 2: символьная ссылка `./public -> ../pissykaka`

Предполагается, что репозиторий с исходниками находится в `../pissykaka`
относительно данного репозитория. Тогда нужно выполнить:

    ln -sfrv ../pissykaka ./public


## 2 Конфиги

Далее нужно подготовить конфиги.

Один из них можно скопировать из примера:

    cp .env.dist .env

Вот конкретное содержимое конфигов с которыми всё работает (не спрашивайте, мне
такое дали, я так делал и оно так работает):

Конфиг `./.env`:

```
#!/bin/bash

MYSQL_DATABASE=test
MYSQL_ROOT_PASSWORD=root
MYSQL_USER=dev
MYSQL_PASSWORD=dev
```

Конфиг `./public/config.php`:

```
<?php

return [
    'db' => [
        'database' => 'test',
        'hostname' => 'db',
        'username' => 'root',
        'password' => 'root'
    ]
];
```

Конфиг `./public/phinx.yml`:

```
paths:
    migrations: migrations
    seeds: seeds

environments:
    default_migration_table: phinxlog
    default_database: development

    development:
        adapter: mysql
        host: db
        name: test
        user: root
        pass: root
        port: 3306
        charset: utf8
        collation: utf8_unicode_ci
```

## 3 Запуск экосистемы

Запускается вся экосистема, чтобы контейнеры с базой данных и бекендом были
доступны - их ещё предстоит настроить, а для этого нужно чтобы он были запущены
и к ним можно было подключиться, запустив шелл.

    docker-compose up

## 4 Созание базы данных

    docker exec -it pissykaka-mysql mysql -proot

Открывается шелл mysql. Нужно ввести команду для создания базы данных test:

    create database if not exists test;

Ответ должен выглядеть примерно вот так:

    Query OK, 1 row affected, 1 warning (0.00 sec)

Чтобы выйти из шелла можно нажать ctrl-d.

## 5 Установка зависимостей и миграция

В запущенном контейнере `pissykaka-php`, выполняется команда для установки зависимостей:

    docker exec -it pissykaka-php composer install

Будет создана директория vendor. Не очень приятно, что ею владеет root, но что
поделать.

Так же в контейнере запускается процедура применения миграций базы
данных:

    docker exec -it pissykaka-php ./vendor/bin/phinx migrate

## 6 Создание доски

Выполняется подключение к шеллу СУБД в запущенном контейнере `pissykaka-mysql`,
чтобы создать доску для тестирования:

    docker exec -it pissykaka-mysql mysql -proot

После выполнения миграций на одном из предыдущих шагов была создана БД `test`,
а называется она так потому что в конфиге так задано. В ней есть по крайней
мере таблицы `boards` (для досок) и `posts` (для постов).  Цель данного пункта
- создать доску с тегом `b` в таблице `boards` в БД `test` и один пост на этой
  доске (одно вхождение в таблице `posts`).

В шелле СУБД нужно выполнить следующую последовательность SQL запросов:

    use test;
    insert into boards values (1, 'b', 'Brat');
    insert into posts values (1, 'Anonymous', '', 'asdf', 1488228666, 1, NULL, 1488228666, 1);


Вместо `'Brat'` можно было бы и по-русски что-нибудь написать, если бы можно
было, конечно. Мне, вот не дано было в этом шелле по-русски писать.

Схема таблиц `posts` и `boards` может быть другой, так что за этим нужно
проследить, если одна из команд выше завершилась с ошибкой.

Чтобы посмотреть схему таблицы `boards` нужно выполнить:

    describe boards;

Чтобы посмотреть схему таблицы `posts` нужно выполнить:

    describe posts;

Только после создания хотя бы одного поста в БД вручную начнёт работать постинг
через API.

Чтобы выйти из шелла БД можно нажать ctrl-d или ctrl-c. Потом то же самое нужно
сделать ещё раз, чтобы покинуть шелл контейнера.

## 7 Проверка

Бекенд `pissykaka` теперь должен быть доступен по адресу `localhost:8888` или
`127.0.0.1:8888`.

Проверить, что доска создалась можно с помощью `curl`:

    curl -v localhost:8888/board/all

Пример вывода результата запроса:

```
*   Trying 127.0.0.1:8888...
* Connected to localhost (127.0.0.1) port 8888 (#0)
> GET /board/all HTTP/1.1
> Host: localhost:8888
> User-Agent: curl/7.72.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.19.3
< Date: Sat, 24 Oct 2020 20:36:27 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
< X-Powered-By: PHP/7.4.11
<
* Connection #0 to host localhost left intact
{"payload":{"boards":[{"id":1,"tag":"b","name":"Brat"}],"posts":[{"id":"2","poster":"Anonymous","subject":"","message":"1234","timestamp":"1603571509","parent_id":null,"tag":"b"},{"id":"1","poster":"Anonymous","subject":"","message":"asdf","timestamp":"1488228666","parent_id":null,"tag":"b"}]},"version":"1.0.0","error":null}
```

Как видно в `boards` имеется вхождение `{"id":1,"tag":"b","name":"Brat"}` - это
только что созданная доска `b`. Среди данных в `posts` так же можно наблюдать
созданный пост.

## 8 Запуск тестов

Нужно выполнить подключение к запущенному контейнеру `pissykaka-php`, и
запустить шелл как в пункте 4.

    docker exec -it pissykaka-php /bin/bash

Далее запустить `phpunit`:

    ./vendor/bin/phpunit
