---
title: "Collections"
product: "Sensu Enterprise Dashboard"
version: "2.11"
description: "Collections are easily sharable groups of items returned by a collection query."
weight: 12
menu: "sensu-enterprise-dashboard-2.11"
---

Collections are a grouping of items returned by a collection query. This query
acts like a global search and it **persists** between the different views.
Collections can be easily **shared** and **saved**.

{{< figure src="/images/enterprise_dashboard/collections/enterprise_dashboard_collections.png" alt="Collections returned by a collection query in the Sensu Enterprise Dashboard" link="/images/enterprise_dashboard/collections/enterprise_dashboard_collections.png" target="_blank" >}}

## Collection Query

The most basic query is composed of a *field* and its *value*, in the form of
`field:value`. A query can use any field, visible or not, to match a value, such
as:

- `dc:us-west-1`
- `subscriber:rabbitmq`
- `subscription:linux`
- `team:webops`

### Regular Expressions

The *match* method of Javascript's *String* object is used to retrieve the
matches, thus the following special characters are available to use exclusively
in the values of a query.

`.` - Matches any single character.
For example, `dc:a.stria` matches the datacenter **austria**.

`*` - Matches the preceding character 0 or more times.
For example, `dc:can*` matches the datacenters **canada** and **vatican**, but
not **cameroon**.

`+` - Matches the preceding character 1 or more times.
For example, `dc:ira+` matches the datacenters **iran** and **iraq**, but not
**ireland**.

`?` - Matches the preceding character 0 or 1 time.
For example, `dc:oc?o` matches the datacenter **cameroon**, but not **morocco**.

`^` - Matches beginning of input.
For example, `dc:^por` matches the datacenter **portugal**, but not
**singapore**.

`$` - Matches end of input.
For example, `dc:nea$` matches the datacenter **guinea**, but not
**guinea-bissau**.


### Operators

The familiar operators, `AND`, `OR`, and `NOT`, are supported. You may use
multiple operators within a single query. Be aware that the use of many
operators may cause higher than normal resource usage.

_NOTE: When using operators, all letters must be capitalized. Operators in lowercase letters are not supported._

#### Examples

`dc:us-east-1 AND name:centos`
Includes all items from the datacenter **us-east-1** that have **centos** in
their name.

`dc:us-east-1 OR dc:us-west-1`
Includes all items from the datacenters **us-east-1** or **us-west-1**.

`dc:us-east-1 NOT name:centos`
Includes all items from the datacenter **us-east-1** that do not have **centos**
in their name.

`NOT dc:us-east-1`
Includes all items that are not part of the datacenter **us-east-1**.

`dc:us-east-1 OR dc:us-west-1 NOT name:centos`
Includes all items from the datacenters **us-east-1** or **us-west-1** that
don't have **centos** in their name.
