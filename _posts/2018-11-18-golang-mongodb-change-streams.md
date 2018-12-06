---
layout: post
title:  "How to use mongo's change stream with golang"
date:   2018-11-18
categories: golang
comments: true
---
Couple days ago I've needed to add MongoDB's change stream support to one of my projects. I found almost no understandable recipes at the StackOverflow and GitHub of how to use it in golang. Take a look at the code I made by myself.

I'm (hope you too) using mgo driver by globalsign to work with MongoDB in Go.

```go
func ChangeStreamWatcher(ctx context.Context) {
    //don't forget to copy *mgo.Session and then close it
    //driver will be able to correctly manage mongo connection sockets
	sess := mgoSession.Copy()
	defer sess.Close()

	//getting access to *mgo.Collection needed
	coll := sess.DB("exampledbname").C("examplecollectionname")

    //connecting to stream
    //Watch() func needed some slice as a first parameter,
    //you can just specify nil, it'll be replaced by []bson.M{} slice
	changeStream, err := coll.Watch(nil, mgo.ChangeStreamOptions{
        BatchSize: 10, 
        //data can come from the stream simultaneously
        //this parameter is like a buffer size
	})
	if err != nil {
		log.WithError(err).Error("Failed to open change stream")
		return //exiting func
	}

	//I want to infinite stream listening
StreamLoop:
	for {
		select {
		case <-ctx.Done(): //if parent context was cancelled
			err := changeStream.Close() //we are closing the stream
			if err != nil {
				log.WithError(err).Error("Change stream closed")
			}
			return //exiting from the func
		default:
			//making a struct for unmarshalling
			changeDoc := struct {
				FullDocument types.YourStruc `bson:"fullDocument"`
			}{}

            //as our change stream can hold multiple elements
            //we need a cycle to process it's data
			for changeStream.Next(&changeDoc) {
				elem := changeDoc.FullDocument
				DoSomethingWithElemFunc(elem)
			}
			//checking the reason of the cycle interruption (why got false at the .Next func)
			err = changeStream.Err()
			if err != nil { // reason in something except coursor depletion
				log.WithError(err).Error("Error while iterating change stream data") //logging
				break StreamLoop                                                     //breaking the loop (func and stream connection to be completely restart)
			}
		}

	}

	//closing stream and exiting from the func
	err = changeStream.Close()
	if err != nil {
		log.WithError(err).Error("Failed to close change stream correctly")
	}
}
```

You can add some workarounds to restart ChangeStreamWatcher func and so on. Take a closer look to the [globalsign's godoc page](https://godoc.org/github.com/globalsign/mgo).

Warning! You need a MongoDB replica set to work with change streams. A quick tip how to create mongo's replica set at your PC [is here](http://37yonub.ru/articles/mongo-replica-set-docker-localhost)