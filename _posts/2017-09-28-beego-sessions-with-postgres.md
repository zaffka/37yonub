---
layout: post
title:  "Beego. How to force its session module to work with Postgres."
date:   2017-09-28 12:15:15
categories: useful_notes
---
If you try to use [official Beego's documentation on Session control](https://beego.me/docs/mvc/controller/session.md)
you'll probably get errors like:
```
[E] [server.go:2619] pq: relation "session" does not exist
OR
[server.go:2619] pq: SSL is not enabled on the server
OR
field: Session.SessionData, unsupport field type , may be miss setting tag
```
here is my quick tip how to fix it.

1) @Postgresql First of all you need to MANUALLY create table "session" in your Postgres database.
```sql
CREATE TABLE session(
    session_key char(64) NOT NULL,
    session_data bytea,
    session_expiry timestamp NOT NULL,
    CONSTRAINT session_key PRIMARY KEY(session_key)
    );
```
Why manually? Why you can't just add a tag to Session struct like this:
```
type Session struct {
    session_key string `orm:"type(char(64))"`
    session_data []byte `orm:"type(bytea)"`
    ...
}
```
and let ORM to deal with the table.

The problem is in this tag `orm:"type(bytea)"`.
Bytes array in MySQL it's a "blob" type, "bytea" from Postgres won't work.
Obviously author of the Beego is using MySQL and made a mistake here.

2) @app.conf According to the documentation you need to use `SessionSavePath` flag to make Postgres work. It's incorrect. 
You should replace this with `SessionProviderConfig`.
For example:
```
SessionProvider = postgresql
SessionProviderConfig = "postgres://username:password@localhost:5432/words?sslmode=disable"
SessionName = session
```
If you work under Linux you maybe don't need to provide `?sslmode=disable` flag.
And pay attention to SessionProvider = postgres**ql** - you need this **ql** letters here.

3) @main.go You have to have this

```
import (
    ...

    "github.com/astaxie/beego/session"
	_ "github.com/astaxie/beego/session/postgres"
	_ "github.com/lib/pq"
)
```
AND something like this

```
func init() {
    ...

	sessionconf := &session.ManagerConfig{
		CookieName: "session",
		Gclifetime: 3600,
	}
	globalSessions, _ := session.NewManager("postgresql", sessionconf)
	go globalSessions.GC()
```

Again, pay attention to `session.NewManager("postgresql"`.
Beego's author have mistyped here. You need this closing **ql**.

4) Finally
![Beego's session module with Postgres](/assets/img/3.png){:class="img-responsive"}
If all is ok, you will have your Beego sessions up and running in Postgres.
And a proper cookie on a client

