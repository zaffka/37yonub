---
layout: post
title:  "Работаем с change streams MongoDB в Golang"
date:   2018-11-18
categories: golang
comments: true
---
Несколько дней назад понадобилось в одном проекте добавить отслеживание изменений в базе с помощью фичи change streams. Фича относительно новая и по её использованию практически нет информации по интернетам. В драйвере монги от globalsign есть поддержка фичи и пример, но он куцый и не даёт общей картинки как что должно работать.

Если вдруг вы столкнулись с необходимость задействовать стрим в своих проектах, возможно кусочек кода ниже поможет разобраться.

```go
func ChangeStreamWatcher(ctx context.Context) {
	//не забываем копировать или клонировать основную сессию монги
	//чтобы драйвер мог правильно управлять пуллом сокетов
	sess := mgoSession.Copy()
	defer sess.Close()

	//получаем доступ к необходимой коллекции (*mgo.Collection)
	coll := sess.DB("exampledbname").C("examplecollectionname")

    //подключаемся к стриму
    //Watch() функция требует слайс в качестве первого параметра,
    //можно просто указать nil, внутри его заменят на слайс []bson.M{}
	changeStream, err := coll.Watch(nil, mgo.ChangeStreamOptions{
        BatchSize: 10, 
		//данные в стрим могут прилетать пачкой,
		//этот параметр позволяет указать, сколько именно объектов прилетит
	})
	defer changeStream.Close()
	if err != nil {
		log.WithError(err).Error("Failed to open change stream")
		return //выходим из функции
	}

	//запускаем бесконечный цикл обработки потока
	for {
		select {
		case <-ctx.Done(): //если родительский контекст закрылся
			err := changeStream.Close() //закрываем стрим
			if err != nil {
				log.WithError(err).Error("Change stream closed")
			}
			return //и выходим из функции
		default:
			//подготавливаем структуру для анмаршалинга данных
			//там прилетает дополнительная информация от монги,
			//поэтому такая странная структура, содержащая FullDocument
			changeDoc := struct {
				FullDocument types.YourStruc `bson:"fullDocument"`
			}{}

			//получаем следующую единицу данных
			ok := changeStream.Next(&changeDoc)

			//если данные не распаковались, то ok == false
			//теперь нам надо вызвать метод Err(), чтобы понять почему
			//если результат вызова метода nil, значит в стриме просто нет данных
			if !ok {
				err := changeStream.Err()
				if err != nil {
					//если ошибка не пустая, выходим из функции
					return
				}
			}

			//если данные распаковались нормально, обрабатываем их
			if ok {
				elem := changeDoc.FullDocument
				DoSomethingWithElemFunc(elem)
			}

		}

	}
}
```

Рекомендую посмотреть код mgo драйвера тут: [globalsign's godoc page](https://godoc.org/github.com/globalsign/mgo) и посмотреть на реализацию функции Watch().

Внимание! Вам понадобится MongoDB в режиме replica set, иначе change streams работать не будет. Про то, как поднять на локальной машине реплику можно [прочитать тут](http://37yonub.ru/articles/mongo-replica-set-docker-rus)