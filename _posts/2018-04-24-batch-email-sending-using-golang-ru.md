---
layout: post
title:  "gRPC-микросервис отправки электронной почты на Golang, ч.1"
date:   2018-04-24
categories: golang grpc
comments: true
---
Сегодня напишем на Go маленький микросервис для рассылки email-сообщений.
Микросервис будет использовать gRPC для клиент-серверного взаимодействия.

Предварительные условия:
* Полный набор настроек для отправки почты посредством SMTP;
* Понимание, что такое переменные окружения и как редактировать bash profile;

В эти детали я углубляться не планирую, гайд расчитан на Linux\Mac пользователя.

Ссылки по теме:
* [Гайд по установке gRPC и Protocol buffers v3](https://grpc.io/docs/quickstart/go.html){:target="_blank"}
* [Описание protocol buffers](https://developers.google.com/protocol-buffers/docs/overview){:target="_blank"}
* [Генераторы protocol buffers v3](https://github.com/google/protobuf/releases){:target="_blank"}
* [Полный код полученного микросервиса](https://github.com/zaffka/newwords-mailer){:target="_blank"}

## Содержание
- [Что такое gRPC](#что-такое-grpc)
- [Необходимые установки](#необходимые-установки)
- [Задача](#задача)
- [Пишем proto-файл](#пишем-proto-файл)
- [Генерируем Go package](#генерируем-go-package)
- [Пишем код микросервиса](#пишем-код-микросервиса)
- [Пишем код клиента](#пишем-код-клиента)

## Что такое gRPC
Совсем коротко - это протокол удалённого вызова процедур с использованием protocol buffers.
А protobuf, в свою очередь - это формат сериализации данных, как XML, только значительно более компактный. По заявлениям google - от 3-х до 10-ти раз компактнее XML и JSON. Работает связка клиент-сервер в gRPC с использованием HTTP/2, за счёт чего в запрсах передаётся существенно меньше служебных данных.
Как результат - очень эффективное взаимодействие клиента и сервера.

Самое важное - нам "из коробки" дают инструментарий для генерации библиотек под любые языки программирования. Ваш клиент на PHP лёгким движением начинает взаимодействовать с вашим сервером на Java. Магия!

## Необходимые установки
...совсем коротко.

1. Качаем [отсюда релиз protocol buffers под нужную платформу](https://github.com/google/protobuf/releases){:target="_blank"}
1. Распаковываем архив, перемещаем исполняемый файл `bin/protoc` в `/usr/local/bin`, а подпапку `include/google` в `/usr/local/include`
1. Добавляем пару строк в `.profile`
```bash
export GOPATH=$HOME/go
export GOBIN=$GOPATH/bin
```
1. Устанавливаем gRPC-пакет для Go `go get -u google.golang.org/grpc`
1. Устанавливаем плагин для генерации библиотеки proto buffers под go `go get -u github.com/golang/protobuf/protoc-gen-go`

## Задача
* Микросервис должен получать сообщения, содержащее два поля - адрес электронной почты (кому шлём письмо) и код.
* Оба поля строкового типа.
* Отвечать на запрос микросервис должен true - когда сообщение принято в очередь на отправку или false - когда очередь переполнена.
* Микросервис должен содержать два метода, для отправки пароля пользователю и для отправки кода восстановления пароля.
Разницы, по сути, нет, просто разные шаблоны писем.
* Код у меня будет распологаться в папке `go/src/github.com/zaffka/newwords-mailer`

## Пишем proto-файл
... он же - описание для данных, которые побегут между нашими клиентом и сервером.

Это один из ключевых моментов. Взяв наш proto-файл, любой программист быстро сгенерит себе библиотеку и сделает клиентскую часть, полностью совместимую с нашим микросервисом. Благодаря понятному формату, можно даже обойтись без документации.

Итак, создадим файл и подпапку `mailer/mailer.proto`
```protobuf
syntax = "proto3"; //указываем версию protocol buffers - третью

//Наш сервис будет называться Mailer и содержать два метода - SendPass и RetrievePass
//Оба метода по сути одинаковы, будут принимать сообщения MsgRequest, на которые ответят MsgReply

service Mailer {
    rpc SendPass(MsgRequest) returns (MsgReply) {}
    rpc RetrievePass(MsgRequest) returns (MsgReply) {}
}

//формат данных для сообщения MsgRequest
//первое поле - строка, название to
//второе поле - строка, название code

message MsgRequest {
    string to = 1;
    string code = 2;
}

//формат данных для сообщения MsgReply
//одно поле - булеан, название sent

message MsgReply {
    bool sent = 1;
}

```

## Генерируем Go package
...содержащий код на основе нашего `mailer/mailer.proto`

`protoc -I mailer/ mailer/mailer.proto --go_out=plugins=grpc:mailer`

В папке проекта, в подпапке `mailer/` появился go package с именем `mailer.pb.go`.

Да, вот так просто мы получили готовый код, покрывающий и клиентскую и серверную части для удалённого вызова процедур.

Просто добавь (в?) include! :)

## Пишем код микросервиса
... но для начала создадим каталоги `serv/` и `client/`, а в них стандартные `main.go`.

Итоговая структура проекта  теперь такая:
```
mailer/
    mailer.proto
    mailer.pb.go
serv/
    main.go
client/
    main.go
```

Открываем `serv/main.go` в любимом редакторе.
Минимальный сервер будет выглядеть примерно так:
```go
package main

import (
	"context"
	"log"
	"net"

	pb "github.com/zaffka/newwords-mailer/mailer"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

//Структура нашего gRPC сервера
type server struct {
}

/*

Методы структуры SendPass и RetrievePass принимают контекст и входящее сообщение,
формат которого мы описали в proto-файле вот так:

message MsgRequest {
    string to = 1;
    string code = 2;
}

Формат ответного сообщения в прото-файле был такой:

message MsgReply {
    bool sent = 1;
}

*/
func (s *server) SendPass(ctx context.Context, in *pb.MsgRequest) (*pb.MsgReply, error) {

    //Код писать не будем, просто ответим true на запрос

	return &pb.MsgReply{Sent: true}, nil 
}

func (s *server) RetrievePass(ctx context.Context, in *pb.MsgRequest) (*pb.MsgReply, error) {

    //А здесь ответим false

	return &pb.MsgReply{Sent: false}, nil
}


func main() {

    //Указываем на каком порту будем слушать запросы
	listener, err := net.Listen("tcp", ":20100")
	if err != nil {
		log.Fatal("failed to listen", err)
	}
	log.Printf("start listening for emails at port %s", ":20100")

    //Создаём новый grpc сервер
	rpcserv := grpc.NewServer()

    //Регистрируем связку сервер + listener
    pb.RegisterMailerServer(rpcserv, &server{})
    reflection.Register(rpcserv)
    
    //Запускаемся и ждём RPC-запросы
	err = rpcserv.Serve(listener)
	if err != nil {
		log.Fatal("failed to serve", err)
	}
}
```

Запускаем сервер `go run serv/main.go`
```console
az@az:~/go/src/github.com/zaffka/newwords-mailer$ go run serv/main.go
2018/04/25 15:51:36 start listening for emails at port :20100
```

## Пишем код клиента
```go
package main

import (
	"context"
	"log"
	"time"

	pb "github.com/zaffka/newwords-mailer/mailer"
	"google.golang.org/grpc"
)

func main() {

    //Открываем соединение, grpc.WithInsecure() означает,
    //что шифрование не используется
	conn, err := grpc.Dial("localhost:20100", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

    /*
    
    Создаём нового клиента, используя соединение conn
    Обратим внимание на название клиента и на название сервиса,
    которое мы определили в proto-файле:

    service Mailer {
    rpc SendPass(MsgRequest) returns (MsgReply) {}
    rpc RetrievePass(MsgRequest) returns (MsgReply) {}
    }

    */
    
    c := pb.NewMailerClient(conn)

    //Определяем контекст с таймаутом в 1 секунду
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

    /*
    Шлём запрос 1, ожидаем получение true в структуру rply
    типа MsgReply, определённую в прото-файле как:

    message MsgReply {
    bool sent = 1;
    }

    */

    rply, err := c.SendPass(ctx, &pb.MsgRequest{"first", "test"})
	if err != nil {
		log.Println("something went wrong", err)
	}
    log.Println(rply.Sent)

    //Шлём запрос 2, ожидаем false
    rply, err = c.RetrievePass(ctx, &pb.MsgRequest{"second", "test"})
	if err != nil {
		log.Println("something went wrong", err)
	}
    log.Println(rply.Sent)
    
}

```

Видим в консоли:
```bash
az@az:~/go/src/github.com/zaffka/newwords-mailer$ go run client/main.go
2018/04/25 16:12:01 true
2018/04/25 16:12:01 false
```

Вот так просто оказывается разделить код на микросервисы, используя Go и инструментарий gRPC.

Часть 2