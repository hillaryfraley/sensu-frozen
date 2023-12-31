---
title: "Clients"
description: "Read this reference documentation for information about Sensu clients, the monitoring agents that are installed and run on every system to be monitored."
product: "Sensu Core"
version: "1.9"
weight: 2
menu:
  sensu-core-1.9:
    parent: reference
---

## Reference documentation

- [What is a Sensu client?](#what-is-a-sensu-client)
- [Client keepalives](#client-keepalives)
  - [What is a client keepalive?](#what-is-a-client-keepalive)
  - [Client registration & the client registry](#registration-and-registry)
    - [Registration and deregistration events](#registration-events)
    - [Proxy clients](#proxy-clients)
  - [How are keepalive events created?](#keepalive-events)
  - [Client keepalive configuration](#client-keepalive-configuration)
- [Client signature](#client-signature)
- [Client subscriptions](#client-subscriptions)
  - [What is a client subscription?](#what-is-a-sensu-subscription)
  - [Round-robin client subscriptions](#round-robin-client-subscriptions)
  - [Client subscription configuration](#client-subscription-configuration)
- [Client socket input](#client-socket-input)
  - [What is the Sensu client socket](#what-is-the-sensu-client-socket)
  - [Example client socket usage](#example-client-socket-usage)
  - [Client socket configuration](#socket-attributes)
- [Standalone check execution scheduler](#standalone-check-execution-scheduler)
- [Managing clients](#managing-clients)
- [Client configuration](#client-configuration)
  - [Example client definition](#example-client-definition)
  - [Client definition specification](#client-definition-specification)
    - [`client` attributes](#client-attributes)
    - [`socket` attributes](#socket-attributes)
    - [`http-socket` attributes](#http-socket-attributes)
    - [`keepalive` attributes](#keepalive-attributes)
    - [`thresholds` attributes](#thresholds-attributes)
    - [`registration` attributes](#registration-attributes)
    - [`deregistration` attributes](#deregistration-attributes)
    - [`ec2` attributes](#ec2-attributes)
    - [`chef` attributes](#chef-attributes)
    - [`puppet` attributes](#puppet-attributes)
    - [`servicenow` attributes](#servicenow-attributes)
    - [`influxdb` attributes](#influxdb-attributes)
    - [`opsgenie` attributes](#opsgenie-attributes)
    - [Custom attributes](#custom-attributes)

## What is a Sensu client?

Sensu clients are [monitoring agents][1], which are installed and run on every
system (e.g. server, container, etc) that needs to be monitored. The client is
responsible for registering the system with Sensu, sending client [keepalive][2]
messages (the Sensu heartbeat mechanism), and executing monitoring checks. Each
client is a member of one or more [subscriptions][3] &ndash; a list of roles
and/or responsibilities assigned to the system (e.g. a webserver, database,
etc). Sensu clients will "subscribe" to (or watch for) [check requests][4]
published by the [Sensu server][5] (via the [Sensu Transport][6]), execute the
corresponding requests locally, and publish the results of the check back to the
transport (to be processed by a Sensu server).

## Client keepalives

### What is a client keepalive?

Sensu Client `keepalives` are the heartbeat mechanism used by Sensu to ensure
that all registered Sensu clients are still operational and able to reach the
[Sensu Transport][6]. Sensu clients publish keepalive messages containing client
configuration data to the Sensu transport every 20 seconds. If a Sensu client
fails to send keepalive messages over a period of 120 seconds (by default), the
Sensu server (or Sensu Enterprise) will create a keepalive [event][7].
Keepalives can be used to identify unhealthy systems and network partitions
(among other things), and keepalive events can trigger email notifications and
other useful actions.

Among those actions is communicating with external sources of truth to determine if the client should still be active. Those sources of truth may be something like a Chef server, a Puppet master, or AWS. This is especially useful when combined with [deregistration events](#registration-events). Adopting this pattern results in clients being deregistered only when a client keepalive threshold has been exceeded. 

### Client registration & the client registry {#registration-and-registry}

In practice, client registrations happens when a Sensu server processes a client
`keepalive` message for a client that is not already registered in the Sensu
client registry (based on the configured client `name` or `source` attribute).
This client registry is stored in the Sensu [data store][8], and is accessible
via the Sensu [Clients API][9].

All Sensu client data provided in client keepalive messages gets stored in the
client registry, which data is used to add context to Sensu [Events][7] and
to detect Sensu clients in an unhealthy state.

#### Registration events

If a [Sensu event handler][30] named `registration` is configured, or if a Sensu
client definition includes a [registration attribute][31], the [Sensu server][5]
will create and process a [Sensu event][7] for the client registration, applying
any configured [filters][26] and [mutators][32] before executing the configured
[handler(s)][30].

Registration events are useful for executing one-time handlers for new Sensu
clients. For example, registration event handlers can be used to update external
[Configuration Management Databases (CMDBs)][34] such as [ServiceNow][35], etc.

To configure a registration event handler, please refer to the [Sensu event
handler documentation][30] for instructions on creating a handler named
`registration`. Alternatively, please see [Client definition `registration`
attributes][31], below.

_WARNING: registration events are not stored in the event registry (in the Sensu
[data store][8]), so they are not accessible via the Sensu API; however, all
registration events are logged in the [Sensu server][5] log._

#### Deregistration events

Similarly to registration events, Sensu can create and process a deregistration event when the Sensu client process stops.
You can use deregistration events to trigger a handler that updates external CMDBs or performs an action to update ephemeral infrastructures.

By default, deregistration events trigger the built-in deregistration extension. To enable deregistration events, set the client `deregister` flag to `true`. This results in the client being immediately depopulated from the client registry in the event that the sensu-client process is administratively stopped. If you require additional logic or actions to be applied when deregistration events are received, specify your custom deregistration event handler using the client [`deregistration` definition scope][48].

_NOTE: To ensure visibility into stopped clients, we recommend implementing a handler inside the keepalive scope to deregister clients based on a keepalive exceeding a threshold. For the vast majority of Sensu use cases, having clients deregister immediately does not produce the desired effect._

#### Proxy clients

Sensu proxy clients (formerly known as "Just-in-time" or "JIT" clients) are
dynamically created clients, added to the client registry if a client does not
already exist for a _check result_.

Proxy client registration differs from keepalive-based registration
because the registration event happens while processing a check result (not a
keepalive message). If a check result includes a `source` attribute, a proxy
client will be created for the `source`, and the check result will be stored
under the newly created client. Sensu proxy clients allow Sensu to monitor
external resources (e.g. on systems and/or devices where a `sensu-client` cannot
be installed, such a network switches), using the defined check `source` to
create a proxy clients for the external resource. Once created, proxy clients
work much in the same way as any other Sensu client; e.g. they are used to store
check execution history and provide context within event data.

_NOTE: `keepalive` monitoring is not supported for proxy clients, as they are
inherently unable to run a `sensu-client`._

##### Proxy client example

Proxy clients are created when a check result includes a `source` attribute. See the example check definition below:

{{< code json >}}
{
  "check": {
    "command": "check-http.rb -u https://sensu.io",
    "subscribers": [
      "demo"
    ],
    "interval": 60,
    "name": "sensu-website",
    "source": "sensuapp.org"
  }
}
{{< /code >}}

_NOTE: This `source` attribute can be provided in a [check definition][14], or
included in a check result published to the Sensu [client input socket][36]._

By default, proxy client data includes a minimal number of attributes. The
following is an example of proxy client data that is added to the registry.

{{< code json >}}
{
  "name": "switch-x",
  "address": "unknown",
  "subscriptions": [],
  "keepalives": false
}
{{< /code >}}

The Sensu API can be used to update proxy client data in the client registry. To
update proxy client data, please refer to the [Client API reference
documentation][9].

##### Create a proxy client via the client socket

The following is an example of how to create a proxy client payload via the [client socket](#client-socket-input), using netcat:

{{< code shell >}}
echo '{"source": "mysql_01", "name": "app_01", "output": "could not connect to mysql", "status": 1}' | nc localhost 3030
{{< /code >}}

### How are keepalive events created? {#keepalive-events}

Sensu servers (including Sensu Enterprise) monitor the Sensu client registry for
stale client data, detecting clients that have failed to send [client keepalive
messages][10] for more than 120 seconds (by default). When a Sensu server
detects that a client hasn't sent a keepalive message within the configured
`threshold`, _the Sensu server (or Sensu Enterprise)_ will create an event (this
is different from how events are created for monitoring checks; see ["How are
Sensu events created?"][11]).

### Client keepalive configuration

For more information on configuring client keepalives, please see the [client
keepalive attribute reference documentation][12] (below).

Sensu client keepalives are published to the Sensu transport every 20 seconds.
Client keepalive behavior can be configured per Sensu client, allowing each
Sensu client to set its own alert thresholds and keepalive event handlers. By
default, client data is considered stale if a keepalive hasn't be received in
`120` seconds (WARNING). By default, keepalive events will be sent to the Sensu
handler named `keepalive` if defined, or the `default` handler will be used.

To configure the keepalive check for a Sensu client, please refer to [the client
`keepalive` attributes reference documentation][12].

## Client signature

Sensu client definitions may specify a unique string identifier as their
signature, which is included in each keepalive message.

Once a client advertises a signature via keepalives, the server will expect that
signature to be present in any keepalives or check results originating from the
client. Any keepalives or check results which do not contain a matching
signature will be dropped with an "invalid client signature" warning in the log.

A client can begin to use a signature if one was not previously configured, but
removing a client signature requires deleting the client.

Providing a unique client signature prevents other clients from accidentally or
maliciously submitting keepalives or check results using the same client name.

## Client subscriptions

### What is a client subscription? {#what-is-a-sensu-subscription}

Sensu's use of the [publish/subscribe pattern of communication][13] allows for
automated registration & de-registration of ephemeral systems. At the core of
this model are Sensu client `subscriptions`.

Each Sensu client has a defined set of subscriptions, a list of roles and/or
responsibilities assigned to the system (e.g. a webserver, database, etc). These
subscriptions determine which monitoring checks are executed by the client.
Client subscriptions allow Sensu to request check  executions on a group of
systems at a time, instead of a traditional 1:1  mapping of configured hosts to
monitoring checks. Sensu checks target Sensu client subscriptions, using the
[check definition attribute `subscribers`][14].

### Round-robin client subscriptions

Round-robin client subscriptions allow checks to be executed on a single client
within a subscription, in a round-robin fashion. To create a round-robin client
subscription, prepend the subscription name with `roundrobin:`, e.g.
`roundrobin:elasticsearch`. Any check that targets the
`roundrobin:elasticsearch` subscription will have its check requests sent to
clients in a round-robin fashion, meaning only one member (client) in the
subscription will execute a roundrobin check each time it is published.

The following is a Sensu client definition that includes a round-robin
subscription.

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "address": "8.8.8.8",
    "subscriptions": [
      "production",
      "webserver",
      "roundrobin:webserver"
    ]
  }
}
{{< /code >}}

The following is a Sensu check definition that targets a round-robin
subscription.

{{< code json >}}
{
  "checks": {
    "web_application_api": {
      "command": "check-http.rb -u https://localhost:8080/api/v1/health",
      "subscribers": [
        "roundrobin:webserver"
      ],
      "interval": 20
    }
  }
}
{{< /code >}}

### Client subscription configuration

To configure Sensu client subscriptions for a client, please refer to [the
client `subscriptions` attribute reference documentation][15].

In addition to the subscriptions defined in the client configuration, Sensu
clients also subscribe automatically to a subscription matching their client
name.  For example, a client named `i-424242` will subscribe to check requests
for the subscription `client:i-424242`. This makes it possible to generate
ad-hoc check requests targeting specific clients via the [`/request` API][49].

## Client socket input

### What is the Sensu client socket?

Every Sensu client has a TCP, UDP and HTTP socket listening for external check result
input. The Sensu client socket(s) listen by default on `localhost` port `3030` for TCP/UDP and
3031 for HTTP and expect JSON formatted check results, allowing external sources (e.g.
your web application, backup scripts, etc.) to push check results without needing to know
anything about Sensu's internal implementation. An excellent client socket use case example
is a web application pushing check results to indicate database connectivity issues.

To configure the Sensu client socket for a client, please refer to [the client
socket attributes][16].


### HTTP Socket

The HTTP socket, just like the TCP and UDP sockets, accepts check results, but it
requires a well-formed HTTP request and exposes other functionality that is not
possible with the raw TCP/UDP sockets. In exchange for a bit more complexity, the
HTTP socket interface has the advantage of being more expressive than a raw TCP/UDP
socket, both in the requests that it accepts and how it responds, and so exposes
more functionality.  The following endpoints are available for the HTTP socket:

`GET /info`

This endpoint returns 200 OK with some basic Sensu information like the version
and transport metrics.

`POST /results`
This endpoint expects an application/json body with a check result

`GET /settings`

This endpoint responds with 200 OK and the sensu configuration. Due to the possible
sensitive nature of the client settings (e.g passwords might be in the config) this
endpoint is protected using HTTP basic authentication and by default the information
returned is redacted (e.g common config keys like 'password' have their values replaced
by 'REDACTED'). See the 'redact' configuration option if you need control over what is
redacted. This endpoint accepts the optional query string ?redacted=false to disable
redaction.

Refer to [the http client socket attributes][50] for details on configuring the HTTP basic
authentication details that this endpoint requires.

`GET /brew`

This endpoint gets you some fresh coffee. Try it!

Any requests for unknown endpoints will get a 404 with some help information in the body.

At the moment only unsecure HTTP (no HTTPS) is supported.

To configure the Sensu HTTP client socket, please refer to [the http client
socket attributes][50].

### Example client socket usage

The following is an example demonstrating external check result input via the
Sensu client TCP socket. The example uses Bash's built-in `/dev/tcp` file to
communicate with the Sensu client socket.

{{< code shell >}}
echo '{"name": "app_01", "output": "could not connect to mysql", "status": 1}' > /dev/tcp/localhost/3030
{{< /code >}}

[Netcat][17] can also be used, instead of the TCP file:

{{< code shell >}}
echo '{"name": "app_01", "output": "could not connect to mysql", "status": 1}' | nc localhost 3030
{{< /code >}}

You can do the same using the HTTP socket:

{{< code shell >}}
curl -v -H "Content-Type: application/json" -X POST -d '{"name": "app_01", "output": "could not connect to mysql", "status": 1}' localhost:3031/results
{{< /code >}}


#### Creating a "dead man's switch"

The Sensu client socket(s) in combination with check TTLs can be used to create
what's commonly referred to as "dead man's switches". Outside of the software
industry, a dead man's switch is a switch that is automatically triggered if a
human operator becomes incapacitated (source: [Wikipedia][18]). Sensu is more
interested in detecting silent failures than incapacitated human operators. By
using Check TTLs, Sensu is able to set an expectation that a Sensu client will
continue to publish results for a check at a regular interval. If a Sensu client
fails to publish a check result and the check TTL expires, Sensu will create an
event to indicate the silent failure. For more information on check TTLs, please
refer to [the check attributes reference documentation][14].

A great use case for the Sensu client socket is to create a dead man's switch
for backup scripts, to ensure they continue to run successfully at regular
intervals. If an external source sends a Sensu check result with a check TTL to
the Sensu client socket, Sensu will expect another check result from the same
external source before the TTL expires.

The following is an example of external check result input via the Sensu client
TCP socket, using a check TTL to create a dead man's switch for MySQL backups.
The example uses a check TTL of `25200` seconds (or 7 hours). A MySQL backup
script using the following code would be expected to continue to send a check
result at least once every 7 hours or Sensu will create an [event][7] to
indicate the silent failure.

{{< code shell >}}
echo '{"name": "backup_mysql", "ttl": 25200, "output": "backed up mysql successfully | size_mb=568", "status": 0}' | nc localhost 3030
{{< /code >}}

{{< code shell >}}
echo '{"name": "backup_mysql", "ttl": 25200, "output": "failed to backup mysql", "status": 1}' | nc localhost 3030
{{< /code >}}

It is also worth noting that you can set the attribute `ttl_status`, which will change the exit code from its default ("1") to a different exit code. You can see an example of this in the check attributes reference documentation linked above.

## Standalone check execution scheduler

In addition to subscribing to [client subscriptions][3] and executing check
requests published by the [Sensu server][19], the Sensu client is able to
maintain its own/separate schedule for [standalone checks][20].

Because the Sensu client shares the same [check scheduling algorithm][21] as the
Sensu server, it is not only possible to have consistency between [subscription
checks][22] and standalone checks &mdash; it's also possible to maintain <abbr
title="typically accurate within 500ms">consistency</abbr> between standalone
checks _across an entire infrastructure_ (assuming that system clocks are
synchronized via [NTP][23]).

## Managing clients

### Delete a client

While shutting down a client stops all check results and keepalives, it doesn't affect the history of the client or remove existing keepalive results.
To delete a client's history from Sensu, use the [Clients API DELETE endpoint][66] or the `x` icon in the dashboard client view.

### Rename a client

To rename a client, modify the [`name` attribute][15] in the client configuration, then restart the client and use the [Clients API DELETE endpoint][66] or the `x` icon in the dashboard client view to remove the history and keepalive results tied to the former client name.
See the appropriate [platform page][67] for instructions on restarting the Sensu client.

## Client configuration

### Example client definition

The following is an example Sensu client definition, a JSON configuration file
located at `/etc/sensu/conf.d/client.json`. This client definition provides
Sensu with information about the system on which it resides. This is a
production system, running a web server and a MySQL database. The client 'name'
attribute is required in the definition, and must be unique.

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "address": "8.8.8.8",
    "subscriptions": [
      "production",
      "webserver",
      "mysql"
    ],
    "socket": {
      "bind": "127.0.0.1",
      "port": 3030
    }
  }
}
{{< /code >}}

### Client definition specification

The client definition uses the `{ "client": {} }` [configuration scope][24].

#### `client` attributes

name         | 
-------------|------
description  | A unique name for the client. The name cannot contain special characters or spaces.
required     | false
default      | System hostname as determined by Ruby `Socket.gethostname`
type         | String
validation   | `/^[\w\.-]+$/`
example      | {{< code shell >}}"name": "i-424242"{{< /code >}}

address      | 
-------------|------
description  | An address to help identify and reach the client. This is only informational, usually an IP address or hostname.
required     | false
default      | Non-loopback IPv4 address as determined by Ruby `Socket.ip_address_list`
type         | String
example      | {{< code shell >}}"address": "8.8.8.8"{{< /code >}}

subscriptions | 
--------------|------
description   |  An array of client subscriptions, a list of roles and/or responsibilities assigned to the system (e.g. webserver). These subscriptions determine which monitoring checks are executed by the client, as check requests are sent to subscriptions. The `subscriptions` array items must be strings.
required      | true
type          | Array
example       | {{< code shell >}}"subscriptions": ["production", "webserver"]{{< /code >}}

safe_mode    | 
-------------|------
description  | If safe mode is enabled for the client. Safe mode requires local check definitions in order to accept a check request and execute the check.
required     | false
type         | Boolean
default      | `false`
example      | {{< code shell >}}"safe_mode": true{{< /code >}}

redact       | 
-------------|------
description  | Client definition attributes to redact (values) when logging and sending client keepalives. When configuring this array, the default values will be overwritten, requiring them to be re-added to your array.
required     | false
type         | Array
default      | {{< code shell >}}[
  "password", "passwd", "pass",
  "api_key", "api_token", "access_key",
  "secret_key", "private_key","secret"
]
{{< /code >}}
example      | {{< code shell >}}"redact": [
  "password",
  "ec2_access_key",
  "ec2_secret_key"
]
{{< /code >}}

socket       | 
-------------|------
description  | The [`socket` definition scope][16], used to configure the [Sensu client socket][36].
required     | false
type         | Hash
example      | {{< code shell >}}"socket": {}{{< /code >}}

keepalives   | 
-------------|------
description  | If Sensu should monitor [keepalives][3] for this client.
required     | false
default      | `true`
type         | Boolean
example      | {{< code shell >}}"keepalives": false{{< /code >}}

keepalive    | 
-------------|------
description  | The [`keepalive` definition scope][12], used to configure [Sensu client keepalives][2] behavior (e.g. keepalive thresholds, etc).
required     | false
type         | Hash
example      | {{< code shell >}}"keepalive": {}{{< /code >}}

signature    | 
-------------|------
description  | Unique string used to authenticate check results from the client in question.
required     | false
type         | String
example      | {{< code shell >}}"signature": "yVNxtPbRGwCYFYEr3V"{{< /code >}}

registration | 
-------------|------
description  | The [`registration` definition scope][31], used to configure [Sensu registration event][37] handlers.
required     | false
type         | Hash
example      | {{< code shell >}}"registration": {}{{< /code >}}

deregister   | 
-------------|------
description  | If a deregistration event should be created upon Sensu client process stop.
required     | false
default      | `false`
example      | {{< code shell >}}"deregister": true{{< /code >}}

deregistration | 
---------------|------
description    | The [`deregistration` definition scope][48], used to configure automated Sensu client de-registration.
required       | false
type           | Hash
example        | {{< code shell >}}"deregistration": {}{{< /code >}}

ec2          | 
-------------|------
description  | The [`ec2` definition scope][38], used to configure the [Sensu Enterprise AWS EC2 integration][39] ([Sensu Enterprise][40] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"ec2": {}{{< /code >}}

chef         | 
-------------|------
description  | The [`chef` definition scope][41], used to configure the [Sensu Enterprise Chef integration][42] ([Sensu Enterprise][40] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"chef": {}{{< /code >}}

puppet       | 
-------------|------
description  | The [`puppet` definition scope][43], used to configure the [Sensu Enterprise Puppet integration][44] ([Sensu Enterprise][40] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"puppet": {}{{< /code >}}

servicenow   | 
-------------|------
description  | The [`servicenow` definition scope][45], used to configure the [Sensu Enterprise ServiceNow integration][46] ([Sensu Enterprise][40] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"servicenow": {}{{< /code >}}

influxdb     | 
-------------|------
description  | The [`influxdb` definition scope][61], used to configure the [Sensu Enterprise InfluxDB integration][62] ([Sensu Enterprise][60] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"influxdb": {}{{< /code >}}

opsgenie     | 
-------------|------
description  | The [`opsgenie` definition scope][63], used to configure the [Sensu Enterprise OpsGenie integration][64] ([Sensu Enterprise][60] users only).
required     | false
type         | Hash
example      | {{< code shell >}}"opsgenie": {}{{< /code >}}

#### `socket` attributes

The following attributes are configured within the `{ "client": { "socket": {} }
}` [configuration scope][24].

##### EXAMPLE {#socket-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "socket": {
      "bind": "127.0.0.1",
      "port": 3030
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#socket-attributes-specification}

enabled      | 
-------------|------
description  | If the Sensu client socket is enabled.
required     | false
type         | Boolean
default      | `true`
example      | {{< code shell >}}"enabled": false{{< /code >}}

bind         | 
-------------|------
description  | The address to bind the Sensu client socket to.
required     | false
type         | String
default      | `127.0.0.1`
example      | {{< code shell >}}"bind": "0.0.0.0"{{< /code >}}

port         | 
-------------|------
description  | The port the Sensu client socket listens on.
required     | false
type         | Integer
default      | `3030`
example      | {{< code shell >}}"port": 3032{{< /code >}}

#### `http socket` attributes

The following attributes are configured within the `{ "client": { "http_socket": {} }
}` [configuration scope][24].

##### EXAMPLE {#http-socket-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "http_socket": {
      "bind": "127.0.0.1",
      "port": 3031,
      "user": "http-auth-user-name",
      "password": "use-somethign-secure-here"
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#http-socket-attributes-specification}

bind         | 
-------------|------
description  | The address to bind the Sensu client socket to.
required     | false
type         | String
default      | `127.0.0.1`
example      | {{< code shell >}}"bind": "0.0.0.0"{{< /code >}}

port         | 
-------------|------
description  | The port the HTTP Sensu client socket listens on.
required     | false
type         | Integer
default      | `3031`
example      | {{< code shell >}}"port": 8000{{< /code >}}

user         | 
-------------|------
description  | The user name to enforce HTTP authentication on endpoints that require it
required     | false
type         | String
example      | {{< code shell >}}"user": "sensu"{{< /code >}}

password     | 
-------------|------
description  | The password to enforce HTTP authentication on endpoints that require it
required     | false
type         | String
example      | {{< code shell >}}"password": "F76639PML6c7sk5nI46N"{{< /code >}}

protect_all_endpoints | 
----------------------|------
description           | If all client HTTP endpoints are protected by HTTP authentication
required              | false
type                  | Boolean
default               | false
example               | {{< code shell >}}"protect_all_endpoints": true{{< /code >}}

#### `keepalive` attributes

The following attributes are configured within the `{ "client": { "keepalive":
{} } }` [configuration scope][24].

##### EXAMPLE {#keepalive-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "keepalive": {
      "handler": "pagerduty",
      "thresholds": {
        "warning": 40,
        "critical": 60
      }
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#keepalive-attributes-specification}

handler      | 
-------------|------
description  | The Sensu event handler (name) to use for events created by keepalives.
required     | false
type         | String
example      | {{< code shell >}}"handler": "pagerduty"{{< /code >}}

handlers     | 
-------------|------
description  | An array of Sensu event handlers (names) to use for events created by keepalives. Each array item must be a string.
required     | false
type         | Array
example      | {{< code shell >}}"handlers": ["pagerduty", "chef"]{{< /code >}}

thresholds   | 
-------------|------
description  | A set of attributes that configure the client keepalive "staleness" thresholds, when a client is determined to be unhealthy.
required     | false
type         | Hash
example      | {{< code shell >}}"thresholds": {}{{< /code >}}

#### `thresholds` attributes (for client keepalives) {#thresholds-attributes}

The following attributes are configured within the `{ "client": { "keepalive": {
"thresholds": {} } } }` [configuration scope][24].

##### EXAMPLE {#thresholds-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "keepalive": {
      "...": "...",
      "thresholds": {
        "warning": 40,
        "critical": 60
      }
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#thresholds-attributes-specification}

warning      | 
-------------|------
description  | The warning threshold (in seconds) where a Sensu client is determined to be unhealthy, not having sent a keepalive in so many seconds._WARNING: keepalive messages are sent at an interval of 20 seconds. Setting a `warning` threshold of 20 seconds or fewer will result in false-positive events. Also note that due to the potential for NTP synchronization issues and/or network latency or packet loss interfering with regular delivery of client keepalive messages, we recommend a minimum `warning` threshold of 40 seconds._
required     | false
type         | Integer
default      | `120`
example      | {{< code shell >}}"warning": 60{{< /code >}}

critical     | 
-------------|------
description  | The critical threshold (in seconds) where a Sensu client is determined to be unhealthy, not having sent a keepalive in so many seconds._WARNING: keepalive messages are sent at an interval of 20 seconds. Setting a `critical` threshold of 20 seconds or fewer will result in false-positive events. Also note that due to the potential for NTP synchronization issues and/or network latency or packet loss interfering with regular delivery of client keepalive messages, we recommend a minimum `critical` threshold of 60 seconds._
required     | false
type         | Integer
default      | `180`
example      | {{< code shell >}}"critical": 90{{< /code >}}

#### `registration` attributes

The following attributes are configured within the `{ "client": {
"registration": {} } }` [configuration scope][24].

##### EXAMPLE {#registration-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "registration": {
      "handler": "servicenow"
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#registration-attributes-specification}

handler      | 
-------------|------
description  | The registration handler that should process the client registration event.
required     | false
type         | String
default      | `registration`
example      | {{< code shell >}}"handler": "registration_cmdb"{{< /code >}}

_NOTE: client `registration` attributes are used to generate [check result][28]
data for the registration [event][7]. Client `registration` attributes are
merged with some default check definition attributes by the [Sensu server][5]
during client registration, so any [valid check definition attributes][14]
&ndash; including [custom check definition attributes][29] &ndash; may be used
as `registration` attributes. The following attributes are provided as
recommendations for controlling client registration behavior._

#### `deregistration` attributes

The following attributes are configured within the `{ "client": {
"deregistration": {} } }` [configuration scope][24].

##### EXAMPLE {#deregistration-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "deregister": true,
    "deregistration": {
      "handler": "deregister_client"
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#deregistration-attributes-specification}

_NOTE: client `deregistration` attributes are used to generate [check
result][28] data for the de-registration event. Client `deregistration`
attributes are merged with some default check definition attributes by the
[Sensu server][5] during client deregistration, so any [valid check definition
attributes][14] &ndash; including [custom check definition attributes][29]
&ndash; may be used as `deregistration` attributes, with the following
exceptions (which are used to ensure the check result is valid): check name,
`output`, `status`, and `issued` timestamp. The following attributes are
provided as recommendations for controlling client deregistration behavior._

handler      | 
-------------|------
description  | The deregistration handler that should process the client deregistration event. By default, Sensu uses the built-in deregistration extension to remove clients from Sensu when a client process stops. To configure the host and port used by the deregistration extension, see the [configuration reference][68].
required     | false
type         | String
default      | `deregistration`
example      | {{< code shell >}}"handler": "cmdb_deregistration"{{< /code >}}

#### `ec2` attributes

The following attributes are configured within the `{ "client": { "ec2": {} }
}` [configuration scope][24].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise AWS EC2 integration][39].**

##### EXAMPLE {#ec2-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "keepalive": {
      "handlers": [
        "ec2"
      ]
    },
    "ec2": {
      "instance_id": "i-424242",
      "account": "sensu-testing",
      "allowed_instance_states": [
        "running",
        "rebooting"
      ]
    }
  }
}{{< /code >}}

_NOTE: In order for the EC2 integration to perform automatic deregistration, the `ec2` handler must be specified in the `keepalive` scope as in the example above._

##### ATTRIBUTES {#ec2-attributes-specification}

instance_id  | 
-------------|------
description  | The AWS EC2 instance ID of the Sensu client system (if different than the [client definition `name` attribute][15]), used to lookup instance status information with the AWS EC2 API.
required     | false
type         | String
default      | defaults to the value of the [client definition `name` attribute][15].
example      | {{< code shell >}}"instance_id": "i-424242"{{< /code >}}

allowed_instance_states | 
------------------------|------
description             | The allowed operational states (e.g. `"running"`) for the instance. If a client keepalive event is created and the EC2 API indicates that the instance is _not_ in an allowed state (e.g. `"terminated"`), Sensu client will be removed from the [client registry][37]. This configuration can be provided to override the [built-in Sensu Enterprise `ec2` integration `allowed_instance_states` configuration][39] for the client.
required                | false
type                    | Array
allowed values          | `pending`, `running`, `rebooting`, `stopping`, `stopped`, `shutting-down`, and `terminated`
default                 | `running`
example                 | {{< code shell >}}"allowed_instance_states": [
  "pending",
  "running",
  "rebooting"
]{{< /code >}}

region         | 
---------------|------
description    | The AWS EC2 region to query for the EC2 instance state(s). This configuration can be provided to override the [built-in Sensu Enterprise `ec2` integration `region` configuration][39] for the client.
required       | false
type           | String
allowed values |
default        | `us-east-1`
example        | {{< code shell >}}"region": "us-west-1"{{< /code >}}

access_key_id | 
--------------|------
description   | The AWS IAM user access key ID to use when querying the EC2 API. This configuration can be provided to override the [built-in Sensu Enterprise `ec2` integration `access_key_id` configuration][39] for the client.
required      | true
type          | String
example       | {{< code shell >}}"access_key_id": "AlygD0X6Z4Xr2m3gl70J"{{< /code >}}

secret_access_key | 
------------------|------
description       | The AWS IAM user secret access key to use when querying the EC2 API. This configuration can be provided to override the [built-in Sensu Enterprise `ec2` integration `secret_access_key` configuration][39] for the client.
required          | true
type              | String
example           | {{< code shell >}}"secret_access_key": "y9Jt5OqNOqdy5NCFjhcUsHMb6YqSbReLAJsy4d6obSZIWySv"{{< /code >}}

timeout      | 
-------------|------
description  | The handler execution duration timeout in seconds (hard stop). This configuration can be provided to override the [built-in Sensu Enterprise `ec2` integration `timeout` configuration][39] for the client.
required     | false
type         | Integer
default      | `10`
example      | {{< code shell >}}"timeout": 30{{< /code >}}

account      | 
-------------|------
description  | The account name as specified in `/etc/sensu/conf.d/ec2.json` if using Sensu across AWS accounts.
required     | false
type         | String
example      | {{< code shell >}}"account": "sensu-testing"{{< /code >}}

#### `chef` attributes

The following attributes are configured within the `{ "client": { "chef": {} }
}` [configuration scope][24].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise Chef integration][42].**

##### EXAMPLE {#chef-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "chef": {
      "node_name": "webserver01"
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#chef-attributes-specification}

node_name    | 
-------------|------
description  | The Chef node name (if different than the [client definition `name` attribute][15]), used to lookup node data in the Chef API.
required     | false
type         | String
default      | defaults to the value of the [client definition `name` attribute][15].
example      | {{< code shell >}}"node_name": "webserver01"{{< /code >}}

endpoint     | 
-------------|------
description  | The Chef Server API endpoint (URL). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `endpoint` configuration][42] for the client.
required     | true
type         | String
example      | {{< code shell >}}"endpoint": "https://api.chef.io/organizations/example"{{< /code >}}

flavor         | 
---------------|------
description    | The Chef Server flavor (is it enterprise?). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `flavor` configuration][42] for the client.
required       | false
type           | String
allowed values | `enterprise`: for Hosted Chef and Enterprise Chef<br>`open_source`: for Chef Zero and Open Source Chef Server
example        | {{< code shell >}}"flavor": "enterprise"{{< /code >}}

client       | 
-------------|------
description  | The Chef Client name to use when authenticating/querying the Chef Server API. This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `client` configuration][42] for the client.
required     | true
type         | String
example      | {{< code shell >}}"client": "sensu-server"{{< /code >}}

key          | 
-------------|------
description  | The Chef Client key to use when authenticating/querying the Chef Server API. This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `key` configuration][42] for the client.
required     | true
type         | String
example      | {{< code shell >}}"key": "/etc/chef/i-424242.pem"{{< /code >}}

ssl_pem_file | 
-------------|------
description  | The Chef SSL pem file use when querying the Chef Server API. This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `ssl_pem_file` configuration][42] for the client.
required     | false
type         | String
example      | {{< code shell >}}"ssl_pem_file": "/etc/chef/ssl.pem"{{< /code >}}

ssl_verify   | 
-------------|------
description  | If the SSL certificate will be verified when querying the Chef Server API. This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `ssl_verify` configuration][42] for the client.
required     | false
type         | Boolean
default      | `true`
example      | {{< code shell >}}"ssl_verify": false{{< /code >}}

proxy_address | 
--------------|------
description   | The HTTP proxy address. This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `proxy_address` configuration][42] for the client.
required      | false
type          | String
example       | {{< code shell >}}"proxy_address": "proxy.example.com"{{< /code >}}

proxy_port   | 
-------------|------
description  | The HTTP proxy port (if there is a proxy). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `proxy_port` configuration][42] for the client.
required     | false
type         | Integer
example      | {{< code shell >}}"proxy_port": 8080{{< /code >}}

proxy_username | 
---------------|------
description    | The HTTP proxy username (if there is a proxy). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `proxy_username` configuration][42] for the client.
required       | false
type           | String
example        | {{< code shell >}}"proxy_username": "chef"{{< /code >}}

proxy_password | 
---------------|------
description    | The HTTP proxy user password (if there is a proxy). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `proxy_password` configuration][42] for the client.
required       | false
type           | String
example        | {{< code shell >}}"proxy_password": "secret"{{< /code >}}

timeout      | 
-------------|------
description  | The handler execution duration timeout in seconds (hard stop). This configuration can be provided to override the [built-in Sensu Enterprise `chef` integration `timeout` configuration][42] for the client.
required     | false
type         | Integer
default      | `10`
example      | {{< code shell >}}"timeout": 30{{< /code >}}

#### `puppet` attributes

The following attributes are configured within the `{ "client": { "puppet": {} }
}` [configuration scope][24].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise Puppet integration][44].**

##### EXAMPLE {#puppet-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "puppet": {
      "node_name": "webserver01"
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#puppet-attributes-specification}

node_name    | 
-------------|------
description  | The Puppet node name (if different than the [client definition `name` attribute][15]), used to lookup node data in PuppetDB.
required     | false
type         | String
default      | defaults to the value of the [client definition `name` attribute][15].
example      | {{< code shell >}}"node_name": "webserver01"{{< /code >}}

#### `servicenow` attributes

The following attributes are configured within the `{ "client": { "servicenow":
{} } }` [configuration scope][24].

**ENTERPRISE: this configuration is provided for using the built-in [Sensu
Enterprise ServiceNow integration][46].**

##### EXAMPLE {#servicenow-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "servicenow": {
      "configuration_item": {
        "name": "webserver01"
      },
      "incident": {
        "product_team": "onboarding"
      }
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#servicenow-attributes-specification}

configuration_item | 
-------------------|------
description        | The [ServiceNow Configuration Item definition scope][45] used to configure the ServiceNow CMDB Configuration Item for the client.
required           | false
type               | Hash
example            | {{< code shell >}}"configuration_item": {
  "name": "webserver01"
}
{{< /code >}}

incident | 
-------------------|------
description        | Key values pairs used to configure ServiceNow incidents for the client. _NOTE: Requires Sensu Enterprise 3.5 or later._
required           | false
type               | Hash
example            | {{< code shell >}}"incident": {
  "product_team": "onboarding"
}
{{< /code >}}

#### `influxdb` attributes

The following attributes are configured within the `{ "client": { "influxdb": {} }
}` [configuration scope][24].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise InfluxDB integration][62].**

##### EXAMPLE {#influxdb-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "influxdb": {
      "tags": {
        "dc": "us-central-1"
      }
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#influxdb-attributes-specification}

tags           | 
---------------|------
description    | Custom tags (key/value pairs) to add to every InfluxDB measurement. Client tags will override any [InfluxDB check tags][65] or [InfluxDB integration tags][62] with the same key.
required       | false
type           | Hash
default        | {{< code shell >}}{}{{< /code >}}
example        | {{< code shell >}}
"tags": {
  "dc": "us-central-1"
}
{{< /code >}}

#### `opsgenie` attributes

The following attributes are configured within the `{ "client": { "opsgenie": {} }
}` [configuration scope][24].

**ENTERPRISE: This configuration is provided for using the built-in [Sensu
Enterprise OpsGenie integration][64].**

##### EXAMPLE {#opsgenie-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "opsgenie": {
      "tags": ["production"]
    }
  }
}
{{< /code >}}

##### ATTRIBUTES {#opsgenie-attributes-specification}

tags         | 
-------------|------
description  | An array of OpsGenie alert tags that will be added to created alerts.
required     | false
type         | Array
default      | `[]`
example      | {{< code shell >}}"tags": ["production"]{{< /code >}}

#### `configuration_item` attributes

The following attributes are configured within the `{ "client": { "servicenow":
{ "configuration_item": {} } } }` [configuration scope][24].

##### EXAMPLE {#configurationitem-attributes-example}

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "...": "...",
    "servicenow": {
      "configuration_item": {
        "name": "webserver01",
        "os_version": "14.04"
      }
    }
  }
}
{{< /code >}}

_PRO TIP: ServiceNow users may provide custom Configuration Item (CI) field values
via the `configuration_item` configuration scope. In this example, the CI field
`os_version` is being set to `14.04`._

##### ATTRIBUTES {#configurationitem-attributes-specification}

name         | 
-------------|------
description  | The [ServiceNow Configuration Item name][47] to be used for the system.
required     | false
type         | String
default      | defaults to the value of the [client definition `name` attribute][15].
example      | {{< code shell >}}"name": "webserver01.example.com"{{< /code >}}

#### Custom attributes

Because Sensu configuration is just [JSON][25] data, it is possible to define
configuration attributes that are not part of the Sensu client specification.
Custom client definition attributes may be defined to provide context about the
Sensu client and the services that run on its system. Custom client attributes
will be included in client [keepalives][2], and [event data][7] and can be used
by Sensu [filters][26] (e.g. only alert on events in the "production"
environment), and accessed via [check token substitution][27].

##### EXAMPLE

The following is an example Sensu client definition that has custom attributes
for the `environment` it is running in, a `mysql` attribute containing
information about a local database, and a link to an operational `playbook`.

{{< code json >}}
{
  "client": {
    "name": "i-424242",
    "address": "10.0.2.101",
    "environment": "production",
    "subscriptions": [
      "production",
      "webserver",
      "mysql"
    ],
    "mysql": {
      "host": "10.0.2.101",
      "port": 3306,
      "user": "app",
      "password": "secret"
    },
    "playbook": "https://wiki.example.com/ops/mysql-playbook"
  }
}
{{< /code >}}

_NOTE: Because client data is included in alerts created by Sensu, custom
attributes that only exist for the purpose of providing troubleshooting
information for operations teams can be extremely valuable._


[?]:  #
[1]:  ../../overview/architecture/#monitoring-agent
[2]:  #client-keepalives
[3]:  #client-subscriptions
[4]:  ../checks/#check-requests
[5]:  ../server
[6]:  ../transport
[7]:  ../events
[8]:  ../data-store
[9]:  ../../api/clients
[10]: #what-is-a-client-keepalive
[11]: ../events/#how-are-events-created
[12]: #keepalive-attributes
[13]: https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
[14]: ../checks#check-definition-specification
[15]: #client-attributes
[16]: #socket-attributes
[17]: http://nc110.sourceforge.net/
[18]: http://en.wikipedia.org/wiki/Dead_man%27s_switch
[19]: ../server/#check-execution-scheduling
[20]: ../checks/#standalone-checks
[21]: ../server/#check-scheduling-algorithm--synchronization
[22]: ../checks/#subscription-checks
[23]: http://www.ntp.org/
[24]: ../configuration#configuration-scopes
[25]: http://www.json.org/
[26]: ../filters
[27]: ../checks/#check-token-substitution
[28]: ../checks/#check-results
[29]: ../checks/#custom-attributes
[30]: ../handlers
[31]: #registration-attributes
[32]: ../mutators
[33]: ../changelog#v0-22-0
[34]: https://en.wikipedia.org/wiki/Configuration_management_database
[35]: https://www.servicenow.com/products/it-operations-management.html
[36]: #client-socket-input
[37]: #registration-and-registry
[38]: #ec2-attributes
[39]: /sensu-enterprise/2.8/integrations/ec2
[40]: https://sensu.io/products/enterprise
[41]: #chef-attributes
[42]: /sensu-enterprise/2.8/integrations/chef
[43]: #puppet-attributes
[44]: /sensu-enterprise/2.8/integrations/puppet
[45]: #servicenow-attributes
[46]: /sensu-enterprise/2.8/integrations/servicenow
[47]: http://wiki.servicenow.com/index.php?title=Introduction_to_Assets_and_Configuration
[48]: #deregistration-attributes
[49]: ../../api/checks#the-request-api-endpoint
[50]: #http-socket-attributes
[60]: /sensu-enterprise/latest
[61]: #influxdb-attributes
[62]: /sensu-enterprise/latest/integrations/influxdb
[63]: #opsgenie-attributes
[64]: /sensu-enterprise/latest/integrations/opsgenie
[65]: ../checks#influxdb-attributes
[66]: ../../api/clients/#clientsclient-delete
[67]: ../../platforms
[68]: ../configuration#top-level-configuration-scopes
