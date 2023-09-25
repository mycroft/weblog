---
title: "Sending/recieving events through Rabbitmq with Pika & Tornado"
date: 2012-10-16T10:02:00
summary: "Quick test of Pika, Tornado & Rabbitmq."
tags:
  - python
---

[**Rabbitmq**](http://www.rabbitmq.com/) is very usefull when moving messages between 2 or more apps.

It's possible to use Rabbitmq with Python by using the [Pika](https://pika.readthedocs.org/en/0.9.13/) module. Pika allows sending messages in synchronous or asynchronous fashions.

When recieving, it's also possible to use Pika, and we'll use [Tornado](http://www.tornadoweb.org/en/stable/) to manage those messages in event mode.


Installing Pika & Tornado:

```sh
$ pip install pika
...
$ pip install tornado
...
```

Following code snippets shows up how to send messages using a producer and to recieve them with a consumer.

Producer
--------

```python
#!/usr/bin/env python

import pika, sys

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

exchange = channel.exchange_declare(exchange='my_exchange', exchange_type='direct')

channel.queue_declare(queue="t1",
                      durable=True,
                      exclusive=False,
                      auto_delete=False)
channel.queue_declare(queue="t2",
                      durable=True,
                      exclusive=False,
                      auto_delete=False)

channel.queue_bind(queue='t1',
                   exchange='my_exchange',
                   routing_key='my_routing_alpha')

channel.queue_bind(queue='t2',
                   exchange='my_exchange',
                   routing_key='my_routing_beta')

for i in range(1):
    message="Hello World!"
    channel.basic_publish(exchange='my_exchange',
                          routing_key='my_routing_beta',
                          body=message)

    print " [x] Sent '%s'" % message

connection.close()
```

Running it:

```sh
$ ./prod.py
 [x] Sent 'Hello World!'
No handlers could be found for logger "pika.adapters.base_connection"
(env)mycroft@dev:~/dev/python-mq$ sudo rabbitmqctl list_queues
Listing queues ...
t2      1
t1      0
...done.
```

Consumer
--------

```python
#!/usr/bin/env python

import tornado
from pika.adapters.tornado_connection import TornadoConnection
import logging
logging.basicConfig()

class Client(object):
    def __init__(self, queue_name = "default"):
        self.queue_name = queue_name

    def connect(self):
        self.connection = TornadoConnection(on_open_callback=self.on_connected)

    def on_connected(self, connection):
        self.connection.channel(self.on_channel_open)

    def on_channel_open(self, channel):
        self.channel = channel
        channel.queue_declare(queue=self.queue_name,
                              durable=True,
                              exclusive=False,
                              auto_delete=False,
                              callback=self.on_queue_declared)

    def on_queue_declared(self, frame):
        self.channel.basic_consume(self.handle_delivery, queue=self.queue_name)

    def handle_delivery(self, channel, method, header, body):
        print channel, method, header
        print body

c = Client('t2')

ioloop = tornado.ioloop.IOLoop.instance()
c.connect()

try:
    ioloop.start()
except:
    c.connection.close()
```

Recieving it:

```sh
$ ./cons.py
<pika.channel.Channel object at 0x1d9a390> <Basic.Deliver(['consumer_tag=ctag1.0', 'redelivered=False', 'routing_key=my_routing_beta', 'delivery_tag=1', 'exchange=my_exchange'])> <BasicProperties>
Hello World!
...
```

For more information, there are an excellent tutorial on the [Rabbitmq website](http://www.rabbitmq.com/tutorials/tutorial-one-python.html).