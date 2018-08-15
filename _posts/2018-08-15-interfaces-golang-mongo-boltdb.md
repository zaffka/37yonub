---
layout: post
title:  "Макетирование(mock) MongoDB, используем MongoDB и BoltDB через единый интерфейс"
date:   2018-08-13
categories: golang
comments: true
---
Продолжаем разговор про интерфейсы в Go. К заметкам [раз](/articles/interfaces) и [два](/articles/interfaces-golang-new) решил добавить [практический пример](https://github.com/zaffka/mongodb-boltdb-mock){:target="_blank"}.  
Представленный [пакет](https://github.com/zaffka/mongodb-boltdb-mock){:target="_blank"} поможет понять, как совместить работу с принципиально разными базами данных в одном пакете и как сделать наш код абсолютно тестируемым - путём добавления, помимо реализаций интерфейса для MongoDB и BoltDB, так называемых mock-структур или макетов.