+++
title = "RabbitMQ"
date = "2016-07-24T14:20:41+08:00"
tags = ["queue","RabbitMQ"]
categories = ["programming"]
menu = ""
banner = "banners/RabbitMQ.png"
+++

Introduction and notes about the `AMQP` product:RabbitMQ, which is a widely used messaging-queue product,Here I will show some concepts about it on this post.
<!--more-->

### enable web interface management
```
rabbitmq-plugins enable rabbitmq_management
```


### Messaging Model.

  ```
  //Simplify the Model
  - A producer is a user application that sends messages.
  - A queue is a buffer that stores messages.
  - A consumer is a user application that receives messages.
  ```
  
  But Actually,The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.Instead, the producer can only send messages to an exchange. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the exchange type.

- default exchange named `amq.direct` is `direct` type exchange, default exchange is very useful,because every queue that is created is automatically bound to it with a routing key which is the same as the queue name.So when we don't declare a exchange,it will work well . the default direct exchange is ideal for the `unicast` routing of messages.Direct exchanges are often used to distribute tasks between multiple workers in a `round-robin` manner. When doing so, it is important to understand that, in AMQP 0-9-1, **messages are load balanced between consumers and not between queues.**

### Exchange Types

    - direct,it can dispatch queues,the exchange will dispatch messages according to the `routing_key` matching,when a exchange is declared,a `routing_key` was defined,after queue binding,the `routing` between queues and exchange was established, when a message goes to the exchange,if the `routing_key` of a message matches the `routing_key` of the exchange,it will be dispatched to the queue,which declares the `routing_key` when it established the binding.
    
    - topic,compared to `direct` type ,`topic` can route based on multiple criteria.
    
    - headers, routing on message headers instead of `routing_key`, Headers exchanges ignore the routing key attribute.
    
    - fanout, it just broadcasts all the messages it receives to all the queues it knows. It ignores that `routing_key` setting on the exchange setting,It just do **mindless broadcasting**.

### Message Publish
  when we do simple messaging things,we don't need to specify the exchange,instead we use an empty string to denote  default or nameless exchange: messages are routed to the queue with the name specified by routing_key, if it exists.

  the exchanges existed can be queried by command like : `rabbitmqctl list_exchanges`

### Bindings
  - tell the exchange to send messages to our queue. That relationship between exchange and a queue is called a binding.
  - `rabbitmqctl list_bindings` to list existing Bindings on the RabbitMQ server.

### Durablity

- queue of rabbitMQ was set durable,but after the RabbitMQ broker was restarted,the queue was empty.Because at the same time  you should set the property called  `DeliveryMode` of the message to be `persistent` too.
  `DeliveryMode` can be set as two values:
    - `1` represents `transient`
    - `2` represents `persistent`,with that value the item will be persisted into disk storage instantly when the broker receives the publishing.
  
### Fair dispatch.
  RabbitMQ just dispatches a message when the message enters the queue. It doesn't look at the number of unacknowledged messages for a consumer. It just blindly dispatches every nth message to the nth consumer.In this context,sometimes, that we push messages evenly to every worker is not properly right.we could dispatch by each worker's capacity,  we can use the `basic.qos` method with the `prefetch_count=${number}` setting. This tells RabbitMQ not to give more than one message to a worker at a time. Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. Instead, it will dispatch it to the next worker that is not still busy.(In a real world.the prefetch_count might be not only one for efficiency,but might not too much either).


### Message Removal
RabbitMQ will remove the messages from memory and queue of course,but if the  consumer dies (its channel is closed, connection is closed, or TCP connection is lost),we will lose all the messages that were dispatched to this particular worker but were not yet handled.
  we should not remove the message from RabbitMQ broker immediately after delivery success to the consumer. Instead the mechanism should be like this:
    1. the messages delivered to consumer,do not remove immediately but wait.
    2. the consumer do all the processing and do ACK to the RabbitMQ broker.
    3. If an ACK was received from the broker ,the message can be removed from broker now.





# Links
  [RabbitMQ concepts](http://www.rabbitmq.com/tutorials/amqp-concepts.html)
