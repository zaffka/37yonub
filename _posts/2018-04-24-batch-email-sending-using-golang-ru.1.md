---
layout: post
title:  "gRPC-микросервис отправки электронной почты на Golang"
date:   2018-04-24
categories: golang grpc
comments: true
---
Сегодня напишем на Go маленький микросервис для рассылки email-сообщений.
Микросервис будет использовать gRPC для общения с клиентами.

Предварительные условия:
* Полный набор настроек для отправки почты посредством SMTP;
* Понимание, что такое переменные окружения и как редактировать bash profile.
В эти детали я углубляться не планирую.

Ссылки по теме:
* [Гайд по установке gRPC и Protocol buffers v3](https://grpc.io/docs/quickstart/go.html){:target="_blank"}
* [Описание protocol buffers](https://developers.google.com/protocol-buffers/docs/overview){:target="_blank"}
* [Генераторы protocol buffers v3](https://github.com/google/protobuf/releases){:target="_blank"}
* [Полный код полученного микросервиса](https://github.com/zaffka/newwords-mailer){:target="_blank"}

## Что такое gRPC
Совсем коротко - это протокол удалённого вызова процедур с использованием protocol buffers.
А protobuf, в свою очередь - это формат сериализации данных, как XML, только значительно более компактный. По заявлениям google - от 3-х до 10-ти раз компактнее XML и JSON. Работает связка клиент-сервер в gRPC с использованием HTTP/2, за счёт чего в запрсах передаётся меньше служебных данных.
Как результат - очень эффективное взаимодействие клиента и сервера.

Самое важное - нам "из коробки" дают инструментарий для генерации библиотек под любые языки программирования. Ваш клиент на PHP лёгким движением начинает взаимодействовать с вашим сервером на Java. В нашем случае с обеих сторон будет, понятно, Go.

Короче, магия! Гугл Дамблдоры, вперёд! :)

## Установка (совсем коротко)
1. Качаем отсюда [релиз protocol buffers под нужную платформу](https://github.com/google/protobuf/releases){:target="_blank"}
1. Распаковываем архив, перемещаем исполняемый файл `bin/protoc` в `/usr/local/bin` (если у вас Linux или Mac OS), а подпапку `include/google` в `/usr/local/include`
1. Добавляем пару строк в `.profile`
```
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
```