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
* [Полный код полученного микросервиса](https://github.com/zaffka/newwords-mailer){:target="_blank"}