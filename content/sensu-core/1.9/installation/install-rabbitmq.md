---
title: "Install RabbitMQ"
description: "This page links to complete instructions for installing and configuring RabbitMQ on Ubuntu/Debian and RHEL/CentOS platforms."
product: "Sensu Core"
version: "1.9"
weight: 7
previous: ../install-redis
menu:
  sensu-core-1.9:
    parent: installation
---

[RabbitMQ][1] is a message bus that [describes itself][2] as _"a
messaging broker - an intermediary for messaging. It gives your applications a
common platform to send and receive messages, and your messages a safe place to
live until received"_. RabbitMQ is also the default [Sensu Transport][3]. When
using RabbitMQ as the Sensu Transport, all Sensu services require access to the
same instance (or cluster) of RabbitMQ to function. **All Sensu users are
encouraged to install and run RabbitMQ on one of the following supported platforms:**

- [Install RabbitMQ on Ubuntu/Debian](../install-rabbitmq-on-ubuntu-debian/)
- [Install RabbitMQ on RHEL/CentOS](../install-rabbitmq-on-rhel-centos/)

_NOTE: please refer to the [Installation Prerequisites documentation][5]
regarding selecting a [Transport][3] for your Sensu installation before
proceeding with RabbitMQ installation. If you intend to use Redis as your
transport, you don't need to install RabbitMQ._

_WARNING: [Sensu Support][4] is available for RabbitMQ installations on
Ubuntu/Debian and RHEL/CentOS operating systems, only._

[1]:  http://www.rabbitmq.com/
[2]:  http://www.rabbitmq.com/features.html
[3]:  ../../reference/transport
[4]:  https://sensuapp.org/support
[5]:  ../installation-prerequisites/#selecting-a-transport
