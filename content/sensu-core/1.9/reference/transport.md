---
title: "Transport"
description: "Read this reference documentation to learn more about the message bus communication provided by the Sensu Transport in Sensu Core."
product: "Sensu Core"
version: "1.9"
weight: 13
menu:
  sensu-core-1.9:
    parent: reference
---

## Reference documentation

- [What is the Sensu Transport?](#what-is-the-sensu-transport)
- [Selecting a transport](#selecting-a-transport)
- [Transport configuration](#transport-configuration)
  - [Example transport definition](#example-transport-definition)
  - [Maximum transport message size](#maximum-transport-message-size)
  - [Transport DNS resolution](#transport-dns)
  - [Transport definition specification](#transport-definition-specification)
    - [Transport attributes](#transport-attributes)

## What is the Sensu Transport?

Sensu services use a message bus (e.g. [RabbitMQ][1]) for communication. This
message bus communication is provided by the [Sensu Transport][2], which is  a
library that makes it possible to leverage alternate transport solutions in
place of RabbitMQ (the default transport). Sensu services requires access to the
same instance of the defined transport (e.g. a RabbitMQ server or cluster) to
function. Sensu check requests and check results are published as “messages” to
the Sensu Transport, and the corresponding Sensu services receive these messages
by subscribing to the appropriate subscriptions.

## Selecting a Transport

The Sensu Transport library makes it possible to replace Sensu's recommended and
default transport (RabbitMQ) with alternative solutions. There are currently
two (2) transports provided with the sensu-transport library: RabbitMQ and
Redis &mdash; each presenting unique performance and functional characteristics.

### The RabbitMQ Transport (recommended)

The RabbitMQ Transport is the original Sensu transport, and continues to be the
recommended solution for running Sensu in production environments.

#### Pros {#rabbitmq-transport-pros}

- Native SSL support
- Pluggable authentication framework
- Support for ACLs

#### Cons {#rabbitmq-transport-cons}

- Adds Erlang as a runtime dependency to the Sensu architecture (only on systems
  where RabbitMQ is running)

### The Redis Transport

Because Sensu already depends on Redis as a data store, using Redis as
a transport greatly simplifies Sensu's architecture by removing the need to
install/configure RabbitMQ and its dependencies. That said, Redis transport
is only considered suitable for simple development environments. **Redis
transport is NOT recommended for production environments!**

#### Pros {#redis-transport-pros}

- Simplifies Sensu architecture by removing need for dedicated transport (by
  using Redis as the data store _and_ transport)
- Comparable or better throughput/performance than RabbitMQ

#### Cons {#redis-transport-cons}

- No native support for SSL
- No support for transport "consumers" metrics (see [Health & Info API][4])
- Not Battle tested: [known bugs][6] in this transport may cause
  unexpected behavior
- Best-effort support only

## Transport configuration

### Example transport definition

The following is an example transport definition, a JSON configuration file
located at `/etc/sensu/conf.d/transport.json`. This example transport
configuration indicates that Redis should be used as the Sensu transport.

{{< code json >}}
{
  "transport": {
    "name": "redis",
    "reconnect_on_error": true
  }
}
{{< /code >}}

### Maximum transport message size

Use `max_message_size` to specify the maximum transport message size (in bytes). Specifying a maximum transport message size helps prevent oversized messages from consuming memory and being persisted to the datastore

The following is an example maximum transport message size definition:

{{< code json >}}
{
  "sensu": {
    "server": {
      "max_message_size": 2097152
    }
  }
}
{{< /code >}}

### Transport DNS resolution {#transport-dns}

The Sensu Transport will resolve provided hostnames before making
connection attempts to the RabbitMQ & Redis transports. Resolving DNS
hostnames prior to connecting allows Sensu to properly handle
resolution failures, log them, and make further attempts to connect to
the selected transport. This also allows Sensu to use DNS as a
transport failover mechanism.

### Transport definition specification {#transport-definition-specification}

The Sensu Transport uses the `"transport": {}` [definition scope][3].

#### Transport attributes {#transport-attributes}

The following attributes are defined within the `"transport": {}`
[definition scope][5].

name           | 
---------------|------
description    | The Transport driver to use.
required       | false
type           | String
allowed values | `rabbitmq`, `redis`
default        | `rabbitmq`
example        | {{< code shell >}}"name": "redis"{{< /code >}}

reconnect_on_error | 
-------------------|------
description        | Attempt to reconnect after a connection error.
required           | false
type               | String
default            | `true`
example            | {{< code shell >}}"reconnect_on_error": "false"{{< /code >}}

[1]:  ../rabbitmq
[2]:  http://github.com/sensu/sensu-transport
[3]:  ../configuration#configuration-scopes
[4]:  ../../api/health-and-info
[5]:  ../../reference/configuration#configuration-scopes
[6]:  https://github.com/sensu/sensu-transport/issues/31
