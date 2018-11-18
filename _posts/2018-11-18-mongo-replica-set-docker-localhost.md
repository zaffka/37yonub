---
layout: post
title:  "How to simply set up Mongo's replica set locally with Docker"
date:   2018-11-18
categories: golang
comments: true
---
Trying to [add MongoDB's change stream support to my Go code](http://37yonub.ru/articles/golang-mongodb-change-streams) I've find out it needs a replica set. Here is a quick tip how to set RS up on your local machine with Docker.

Before all, you'll need the docker compose installed at your PC.  
Check how to install it [here](https://docs.docker.com/compose/install/){:target="_blank"}

#### First step
Make a compose file `docker-compose.yml` with a following content:
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

#### Up containers
Now just say `docker-compose up -d` to start three instances of the MongoDB.
Compose will make a network for these three containers so we don't need to deal with this by ourself.

Look at the ports - each container exposes port from 27017(standart) to 27019 to the outer world. In our case outer - it's your local machine.  
 From this moment you can connect to every instance of MongoDB inside Docker's network.

Now let's make our three containers to wark as a replica set.

#### Set up replica set
Connect to mongo shell at our primary container.  
Say `docker exec -it mongo0 mongo`.  

Type inside a shell 

1. `config={"_id":"rs0","members":[{"_id":0,"host":"mongo0:27017"},{"_id":1,"host":"mongo1:27017"},{"_id":2,"host":"mongo2:27017"}]}`
2. rs.initiate(config)

And that's it.

If you connect to each container, you'll see 
`rs0:PRIMARY>` prompt at mongo0, and `rs0:SECONDARY>` at others two.

Now you can connect your code to replica set using a DSN string like this:  
`mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0`