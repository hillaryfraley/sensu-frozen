---
title: "Aggregates"
description: "Read this reference documentation for information about Sensu named aggregates, collections of multiple check results you can treat as a single result."
product: "Sensu Core"
version: "1.9"
weight: 4
menu:
  sensu-core-1.9:
    parent: reference
---
## Reference documentation

- [What is a Sensu named aggregate?](#what-is-a-check-aggregate)
  - [When should named aggregates be used?](#when-should-check-aggregates-be-used)
- [How do named aggregates work?](#how-do-check-aggregates-work)
  - [Example aggregated check result](#example-aggregated-check-result)
- [Aggregate configuration](#aggregate-configuration)
  - [Example aggregate definition](#example-aggregate-definition)
  - [Aggregate definition specification](#aggregate-definition-specification)
    - [Aggregate `check` attributes](#aggregate-check-attributes)

## What is a Sensu named aggregate? {#what-is-a-check-aggregate}

Sensu named aggregates are collections of [check results][1], accessible via
the [Aggregates API][2]. Check aggregates make it possible to treat the results
of multiple disparate check results &ndash; executed across multiple disparate
systems &ndash; as a single result.

### When should named aggregates be used? {#when-should-check-aggregates-be-used}

Check aggregates are extremely useful in dynamic environments and/or
environments that have a reasonable tolerance for failure. Check aggregates
should be used when a service can be considered healthy as long as a minimum
threshold is satisfied (e.g. are at least 5 healthy web servers? are at least
70% of N processes healthy?).

## How do named aggregates work? {#how-do-check-aggregates-work}

Check results are included in an aggregate when a check definition includes the
[`aggregate` definition attribute][3]. Check results that provide an
`"aggregate": "example_aggregate"` are aggregated under the corresponding name
(e.g. `example_aggregate`), effectively capturing multiple check results as a
single aggregate.

### Example aggregated check result

Aggregated check results are available from the [Aggregates API][2], via the
`/aggregates/:name` API endpoint. An aggregate check result provides a
set of counters indicating the total number of client members, checks, and
check results collected, with a breakdown of how many results were recorded per
status (i.e. `ok`, `warning`, `critical`, and `unknown`).

{{< code json >}}
{
  "clients": 15,
  "checks": 2,
  "results": {
    "ok": 18,
    "warning": 0,
    "critical": 1,
    "unknown": 0,
    "total": 19,
    "stale": 0
  }
}
{{< /code >}}

Additional aggregate data is available from the Aggregates API, including Sensu
client members of a named aggregate, and the corresponding checks which are
included in the aggregate:

{{< code shell >}}
$ curl -s http://localhost:4567/aggregates/elasticsearch/clients | jq .
[
  {
    "name": "i-424242",
    "checks": [
      "elasticsearch_service",
      "elasticsearch_cluster_health"
    ]
  },
  {
    "name": "1-424243",
    "checks": [
      "elasticsearch_service"
    ]
  },
]
{{< /code >}}

Aggregate data may also be fetched per check that is a member of the named
aggregate, along with the corresponding clients that are producing results for
said check:

{{< code shell >}}
$ curl -s http://localhost:4567/aggregates/elasticsearch/checks | jq .
[
  {
    "name": "elasticsearch_service",
    "clients": [
      "i-424242",
      "i-424243"
    ]
  },
  {
    "name": "elasticsearch_cluster_health",
    "clients": [
      "i-424242"
    ]
  }
]
{{< /code >}}

## Aggregate configuration

### Example aggregate definition

The following is an example [check definition][6], a JSON configuration file located at `/etc/sensu/conf.d/check_aggregate_example.json`.

{{< code shell >}}
{
  "checks": {
    "example_check_aggregate": {
      "command": "do_something.rb -o option",
      "aggregate": "example_aggregate",
      "interval": 60,
      "subscribers": [
        "my_aggregate"
      ],
      "handle": false
    }
  }
}{{< /code >}}

### Aggregate definition specification

_NOTE: aggregates are created via the [`aggregate` Sensu `check` definition
attribute][4]. The configuration example(s) provided above, and the
"specification" provided here are for clarification and convenience only (i.e.
this "specification" is just a subset of the [check definition
specification][5], and not a definition of a distinct Sensu primitive)._

#### Aggregate `check` attributes

aggregate    | 
-------------|------
description  | Create a named aggregate for the check. Check result data will be aggregated and exposed via the [Sensu Aggregates API][2].
required     | false
type         | String
example      | {{< code shell >}}"aggregate": "elasticsearch"{{< /code >}}

aggregates   | 
-------------|------
description  | An array of strings defining one or more named aggregates (described above).
required     | false
type         | Array
example      | {{< code shell >}}"aggregates": [ "webservers", "production" ]{{< /code >}}


handle       | 
-------------|------
description  | If events created by the check should be handled.
required     | false
type         | Boolean
default      | true
example      | {{< code shell >}}"handle": false{{< /code >}}_NOTE: although there are cases when it may be helpful to aggregate check results **and** handle individual check results, it is typically recommended to set `"handle": false` when aggregating check results, as the [purpose of the aggregation][8] should be to act on the state of the aggregated result(s) rather than the individual check result(s)._

[1]:  ../checks#check-results
[2]:  ../../api/aggregates
[3]:  ../checks#check-definition-specification
[4]:  ../checks#check-attributes
[5]:  ../checks#check-definition-specification
[6]:  ../checks#check-configuration
[7]:  ../checks#standalone-checks
[8]:  #when-should-check-aggregates-be-used
[9]:  ../checks#how-are-checks-scheduled
