---
layout: post
title:  "Быстрый способ поднять реплика сет MongoDB"
date:   2018-11-18
categories: golang
comments: true
---
Разбирался как [использовать change stream от MongoDB в Go](http://37yonub.ru/articles/golang-mongodb-change-streams-rus), обнаружил, что для использования фичи, нужна монго в режиме репликации.
Вот как можно быстро поднять реплика сет на локальном компьютере в докер-контейнерах. 

Надеюсь docker-compose уже установлен? Если нет, то дока [тут](https://docs.docker.com/compose/install/){:target="_blank"}

#### Первый шаг
Создадим компоуз-файл `docker-compose.yml` с таким вот содержимым:
```yml
version: "3"
services:
  mongo0:
    hostname: mongo0
    container_name: mongo0
    image: mongo
    ports:
      - 27017:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0" ]
  mongo1:
    hostname: mongo1
    container_name: mongo1
    image: mongo
    ports:
      - 27018:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0" ]
  mongo2:
    hostname: mongo2
    container_name: mongo2
    image: mongo
    ports:
      - 27019:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0" ]
```

#### Запускаем контейнеры
Говорим `docker-compose up -d`. Компоуз поднимет для нас три контейнера, объединив их в докер-сеть, так что об этом нам беспокоиться не нужно.

Обратим внимание на открытые порты. Внутри сети это стандартный 27017, чтобы монга могла сконфигурить репликацию. А вот вовне - на локальную машину открыты разные порты c 17 по 19-й, чтобы наш код мог подцепить все три инстанса.

Теперь надо заставить три экземпляра mongodb работать в режиме репликации.  
Это просто.

#### Настраиваем replica set
Заходим в монго шелл на первом контейнере `docker exec -it mongo0 mongo`.  

Печатаем...

1. `config={"_id":"rs0","members":[{"_id":0,"host":"mongo0:27017"},{"_id":1,"host":"mongo1:27017"},{"_id":2,"host":"mongo2:27017"}]}`
2. rs.initiate(config)

И всё :)

Теперь, если поочерёдно подключиться к каждому из контейнеров, мы увидим приглашение 
`rs0:PRIMARY>` в mongo0, и `rs0:SECONDARY>` на двух оставшихся.

Можно теперь подключаться из нашего кода к репликам с такой вот ресурсной строкой:  
`mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0`