---
title: "Prerequisites"
description: "Read this page to learn about installing Sensu's dependencies, the data store and transport, in a standalone configuration."
weight: 3
product: "Sensu Core"
version: "1.9"
next: ../install-sensu-server-api
previous: ../installation-strategies
menu:
  sensu-core-1.9:
    parent: installation
---

As mentioned earlier in this guide, Sensu's [architecture][1] is one
of its most compelling features. While this modular design makes it infinitely
adaptable to monitor any infrastructure, it also depends on some external
services to function:

- [Data Store (Redis)](#data-store)
- [Transport](#transport)

_NOTE: this guide will focus on installing Sensu's dependencies in a
[standalone][2] configuration. However, in a live/production environment, it is
strongly recommended that &ndash; at minimum &ndash; the Sensu data store and
transport run on a different system than the Sensu server and API (i.e. in a
[distributed][3] or [high availability][4] configuration). Please review the
Sensu [Installation Strategies][5] for more information._

## Data Store (Redis) {#data-store}

Sensu uses [Redis][6] as a [data-store][7] for storing persistent data such as the
client registry and check results. Two Sensu Core services, the server and API
require access to the same instance of Redis to function (i.e. the Sensu client
does not communicate with Redis).

## Transport

Sensu services use a message bus (e.g. [RabbitMQ][8]) for communication. This
message bus communication is provided by the [Sensu Transport][9], which is a
library that makes it possible to leverage alternate transport solutions in
place of RabbitMQ (the default transport). Sensu services requires access to the
same instance of the defined transport (e.g. a RabbitMQ cluster) to  function.
Sensu check requests and check results are published as "messages" to  the Sensu
Transport, and the corresponding Sensu services receive these messages  by
subscribing to the appropriate subscriptions.


[1]:  ../../overview/architecture
[2]:  ../installation-strategies/#standalone
[3]:  ../installation-strategies/#distributed
[4]:  ../installation-strategies/#high-availability
[5]:  ../installation-strategies
[6]:  http://redis.io
[7]:  ../../reference/data-store
[8]:  ../../reference/rabbitmq
[9]:  ../../reference/transport
