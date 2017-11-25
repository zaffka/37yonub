---
layout: post
title:  "Как научить модуль Session из Beego работать с Postgresql"
date:   2017-09-28 11:15:15
categories: golang
comments: true
---
Процесс изучения Go движется. Уже даже написал пару небольших программулин на заказ - друзьям и родственникам, а также начал свой домашний проект. Для проекта был выбран фреймворк Beego и база Postgresql. К сожалению, проблемы не заставили себя ждать. В этой заметке решение одной из них.

## Модуль сессий.

Если почитать [официальную документацию Beego](https://beego.me/docs/mvc/controller/session.md) на этот модуль (читать придётся на английском) и попробовать начать его использовать, с большой долей вероятности увидишь одну из этих ошибок:
```
[E] [server.go:2619] pq: relation "session" does not exist
ИЛИ
[server.go:2619] pq: SSL is not enabled on the server
ИЛИ
field: Session.SessionData, unsupport field type , may be miss setting tag
```
Дело в том, что автор Beego - китаец, он допускает довольно много опечаток и ошибок в английских текстах.


1) @Postgresql

Первое, что нужно сделать, чтобы правильно завести сессии с хранением в Postgres - это ВРУЧНУЮ создать табличку в базе.
```sql
CREATE TABLE session(
    session_key char(64) NOT NULL,
    session_data bytea,
    session_expiry timestamp NOT NULL,
    CONSTRAINT session_key PRIMARY KEY(session_key)
    );
```
Почему вручную, а не связкой со структурой через ОРМ?
Давайте смотреть. Вот возможная модель для сессий.
```
type Session struct {
    session_key string `orm:"type(char(64))"`
    session_data []byte `orm:"type(bytea)"`
    ...
}
```
Проблема в теге `orm:"type(bytea)"`.

В работе автор Beego, очевидно, использует MySQL, в котором бинарный тип - это "blob", а в Postgres это "bytea", поддержку которого просто забыли добавить. 

2) @app.conf

В документации предлагается указывать параметр `SessionSavePath`, чтобы сконфигурить соединение с базой. С ним, как вы понимаете, соединение не устанавливается.

Нужно заменить `SessionSavePath` на `SessionProviderConfig`.

Например так:
```
SessionProvider = postgresql
SessionProviderConfig = "postgres://username:password@localhost:5432/words?sslmode=disable"
SessionName = session
```
Если вы работаете под Линукс, параметр `?sslmode=disable` вам, вероятно, не понадобится.

И обратите внимание на SessionProvider = postgres**ql** - вот эти две последние буквы  **ql** в документации отсутствуют. Мелочь, а можно залипнуть.

3) @main.go

Здесь нужно сделать так:

```
import (
    ...

    "github.com/astaxie/beego/session"
	_ "github.com/astaxie/beego/session/postgres"
	_ "github.com/lib/pq"
)
```

4) После этих шагов контроль сессий должен заработать.
![Beego's session module with Postgres](/assets/img/3.png){:class="img-responsive"}
Если всё хорошо, вы увидите, что в нашу табличку в БД начали добавляться записи о сессиях, а на клиенте пишется правильное cookie.

Search tags: _Go_, _Golang_, _Beego_, _Postgresql_
