---
layout: post
title:  "gRPC-микросервис отправки электронной почты на Golang, ч.2"
date:   2018-04-25
categories: golang grpc
comments: true
---
В [первой части статьи](/articles/batch-email-sending-using-golang-ru) мы подняли простейший микросервис, используя инструментарий gRPC.
Во второй части разберёмся, как отправлять письма из Go, использовать для этого защищённое соединение и делать массовые рассылки.

## Содержание
- [Как работает SMTP](#как-работает-smtp)

## Как работает SMTP
Для наглядности возьмём из википедии пример простейшей SMTP-сессии.
```
S: (ожидает соединения)
C: (Подключается к порту 25 сервера)
S:220 mail.company.tld ESMTP is glad to see you!
C:HELO
S:250 domain name should be qualified
C:MAIL FROM: <someusername@somecompany.ru>
S:250 someusername@somecompany.ru sender accepted
C:RCPT TO: <user1@company.tld>
S:250 user1@company.tld ok
C:RCPT TO: <user2@company.tld>
S:550 user2@company.tld unknown user account
C:DATA
S:354 Enter mail, end with "." on a line by itself
C:From: Some User <someusername@somecompany.ru>
C:To: User1 <user1@company.tld>
C:Subject: tema
C:Content-Type: text/plain
C:
C:Hi!
C:.
S:250 769947 message accepted for delivery
C:QUIT
S:221 mail.company.tld CommuniGate Pro SMTP closing connection
S: (закрывает соединение)
```

Видно, что, после установки соединения, начинается текстовый диалог клиента и сервера, предваряемый командой HELO(может быть EHLO - если используется более продвинутая версия SMTP-протокола), а после команды `DATA` идёт передача непосредственно тела письма.

Важный момент - в рамках одного такого диалога можно последовательно отправить серию сообщений. 

Второй важный момент - большинство современных почтовых серверов, работают с шифрованием. Поэтому пример отправки сообщения с plaintext-авторизацией, который есть [на сайте golang.org в описании smtp-пакета](https://golang.org/pkg/net/smtp/#example_SendMail){:target="_blank"}, нам не подходит.

## Задача
Учитывая вышесказанное, наш микросервис должен:
* Устанавливать соединение, используя tls.
* Слать несколько сообщений за одну сессию.
* Переподключаться в случае закрытия соединения.
* Отбивать новые запросы от клиентов, если очередь на отправку почты во внешний мир переполнена.

## Решение

(I)
Для начала определим ряд глобальных типов, переменных и констант, содержащих:
* Названия шаблонов писем
* Параметры конфигурации микросервиса (type conf struct)
* ...в том числе параметры соединения с smtp-сервером
* Переменную cnf, содержащую заполненную структуру conf
* Переменную tpl, содержащую шаблоны писем
* Переменную queue - канал, содержащий данные типа Message

```go
const (
	passtpl     = "password.msg"
	retrievetpl = "retrieve.msg"
)

type conf struct {
	smtphost, user, from, servicename, pass, serveport string
}

var cnf conf               
var tpl *template.Template 
var queue chan Message     

```
(II)
Инициализируем наши глобальные переменные:
```go
func init() {
	cnf = conf{
		os.Getenv("MAILER_REMOTE_HOST"),
		os.Getenv("MAILER_USER"),
		os.Getenv("MAILER_FROM"),
		os.Getenv("MAILER_SERVICENAME"),
		os.Getenv("MAILER_PASSWORD"),
		os.Getenv("MAILER_SERVE_PORT"),
	}
	tpl = template.Must(template.New("").ParseGlob("./templates/mail/*.msg"))

	queue = make(chan Message, 10)
}
```
Видим, что переменная cnf теперь указывает на структуру conf, которая заполнена данными из переменных окружения.

Конфигурация будет содержать адрес smtp-сервера (в моём случае smtp.yandex.ru:465), логин, пароль для доступа, текстовое название сервиса от имени которого будет идти рассылка, адрес from и последним - сервисный порт, на котором микросервис будет отвечать на запросы от клиентов локальной сети.

В ч.1 порт был указан явно - 20100-й.

Здесь, по-хорошему, надо добавить проверку того, что параметры конфигурации не пусты.

Дальше читаем в переменную tpl из каталога `templates/mail/` все файлы типа `*.msg`, содержащие шаблоны писем. Парсинг обёрнут в template.Must - приложение запаникует, если шаблонов не будет на месте.

И последняя переменная queue - это наша очередь сообщений. Создаём канал типа Message на 10 элементов.

(III)
Определяем тип Message и один метод к нему, который, на основании соответствующего шаблона сообщения, построит нам тело письма (то, что будет отправлено после команды DATA в smtp-сессии).

```go
type Message struct {
	From, To, Code, tplname string
}

//метод возвращает срез байт
func (m *Message) getMailBody() []byte {
	buf := new(bytes.Buffer)
	err := tpl.ExecuteTemplate(buf, m.tplname, m)
	if err != nil {
		log.Println(err)
	}
	return buf.Bytes()
}
```

(IV)
Модифицируем нашу структуру и методы для обработки rpc-запросов.
Метод приведу только один, нам главное содержимое.
```go
type server struct {
}

func (s *server) SendPass(ctx context.Context, in *pb.MsgRequest) (*pb.MsgReply, error) {

    //В переменную m считываем MsgRequest(смотрим в mail.proto, чтобы вспомнить, что это).

	m := Message{From: fmt.Sprintf("%s <%s>", cnf.servicename, cnf.from), To: in.To, Code: in.Code, tplname: passtpl}

    //А вот дальше нам надо записать полученное сообщение в очередь.
    //Сделать нам это нужно в неблокирующем стиле, для этого используем select.

	select {
        case queue <- m: //Пишем в канал, если он заблокирован, выполняем default-ветку
    
        default:
        //Отвечаем на rpc-запрос false-ем
        //Таким образом клиент узнает, что сообщение по каким-то причинам не принято и сможет обработать ситуацию
	return &pb.MsgReply{Sent: false}, nil
	}

    //Ну, а если все хорошо,  отвечаем клиенту true
    return &pb.MsgReply{Sent: true}, nil
}
```
Вот таким образом мы будем наполнять очередь сообщений queue и отвечать клиентам false-ами, в случае её переполнения.

## Отправка почты
Для непосредственной работы по отправке почты, реализуем две функции.

Во-первых, функцию `getSMTPClient()`, отвечающую за установку соединения и поднимающую smtp-клиента для работы с удалённым хостом.

Этой функцией мы сначала открываем защищённое tls соединение - вызываем метод tls.Dial
И только после этого, внутри tls-коннекта, создаём smtp-клиента.

```go
func getSMTPClient() *smtp.Client {
	var err error
	host, _, _ := net.SplitHostPort(cnf.smtphost)

	tlsconfig := &tls.Config{
		InsecureSkipVerify: true,
		ServerName:         host,
	}

	conn, err := tls.Dial("tcp", cnf.smtphost, tlsconfig)
	if err != nil {
		log.Println("tls.dial", err)
	}

	client, err := smtp.NewClient(conn, cnf.smtphost)
	if err != nil {
		log.Println("new client", err)
	}

	auth := smtp.PlainAuth("", cnf.user, cnf.pass, cnf.smtphost)

	if err = client.Auth(auth); err != nil {
		log.Println("auth", err)
	}

	return client
}
```
Во-вторых, нашу основную функцию-цикл `messageLoop()`, которая будет досылать сообщения в smtp-сессию, пока открыт канал и есть сообщения в очереди queue.
```go
func messageLoop() {
    //Инициализируем smtp-клиента
    //вызовом функции getSMTPClient()

	client := getSMTPClient()
	defer client.Quit()

    //Начинаем читать в бесконечном цикле данные из канала queue
    //и писать их последовательно в smtp-клиента
	for m := range queue {

		err := client.Noop()
		if err != nil {
			log.Println("reestablish connection", err)
			client = getSMTPClient()
		}

		if err = client.Mail(cnf.user); err != nil {
			log.Println(err)
		}

		if err = client.Rcpt(m.To); err != nil {
			log.Println(err)
		}

		writecloser, err := client.Data()
		if err != nil {
			log.Println(err)
		}

		_, err = writecloser.Write(m.getMailBody())
		if err != nil {
			log.Println(err)
		}

		err = writecloser.Close()
		if err != nil {
			log.Println(err)
		}

	}

}
```
Самый **важный момент** в коде выше вот этот:
```go
		err := client.Noop()
		if err != nil {
			log.Println("reestablish connection", err)
			client = getSMTPClient()
		}
```
Данный **вызов идёт в начале каждого цикла**. Наш микросервис, а вернее его SMTP-клиент, пробует считывать что-либо с удалённого почтового сервера.
Если чтение не удаётся, мы получаем ошибку и переустанавливаем соединение с smtp-сервером вызовом функции `getSMTPClient()`.

Метод smtp.Noop(), по сути, такой ping для smtp-сессии.

Обрабатывая его, **мы обеспечиваем восстановление соединения для отправки следующей серии сообщений**.

Ну и нам остаётся только добавить нашу "петлю" messageLoop() в функцию main.
```go
func main() {
    go messageLoop()
    
    ...остальной код...
```

## Результат работы

Запускаем клиента, который модифицирован на отправку 5 сообщений.
```bash
az@az:~/go/src/github.com/zaffka/newwords-mailer$ go run client/main.go
2018/04/25 21:17:48 true
2018/04/25 21:17:48 true
2018/04/25 21:17:48 true
2018/04/25 21:17:48 true
2018/04/25 21:17:48 true
```

И вот они, в почтовом ящике на Яндексе.
![gRPC Golang batch emails](/assets/img/batchemails.png){:class="img-responsive"}

Обратите внимание, письма идут вразнобой - это потому, что на клиенте была симулирована параллельная отправка RPC-запросов к нашему микросервису.

В качестве самостоятельной работы, попробуйте поиграть(уменьшить time.Second до, например, time.Nanosecond) с context.WithTimeout в файле `client/main.go` и посмотрите, как покажет себя наш код, в случае слишком долгого выполнения запроса.

На этом всё.
[Полный код проекта](https://github.com/zaffka/newwords-mailer){:target="_blank"} выложен на гитхаб.