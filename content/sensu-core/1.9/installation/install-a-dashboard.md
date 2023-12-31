---
title: "Sensu Dashboards"
description: "This page provides an overview and links to further information about the two Sensu dashboard solutions: Uchiwa and Sensu Enterprise Dashboard."
weight: 4.5
product: "Sensu Core"
version: "1.9"
next: ../install-redis
previous: ../install-sensu-client
menu:
  sensu-core-1.9:
    parent: installation
---

Sensu was originally designed as an API-based monitoring solution, enabling
operations teams to compose monitoring solutions where Sensu provides the
monitoring instrumentation, collection of telemetry data, scalable event
processing, comprehensive APIs &ndash; _and plugins for sending data to
dedicated dashboard solutions_. However, as the Sensu Core project and community
have matured, the need for an optional Sensu dashboard has become more obvious.
As a result, there are now two (2) dashboard solutions for Sensu: **Uchiwa**
(for Sensu Core users), and the **Sensu Enterprise Dashboard** (for [Sensu
Enterprise][sensu-enterprise] customers).

Both Uchiwa and the Sensu Enterprise Dashboard work by accessing data directly
via the Sensu APIs (i.e. the [Sensu Core APIs][core-api], or the [Sensu
Enterprise APIs][enterprise-api]).

- [Install Sensu Enterprise Dashboard on Ubuntu/Debian](../../platforms/sensu-on-ubuntu-debian/#sensu-enterprise)
- [Install Sensu Enterprise Dashboard on RHEL/CentOS](../../platforms/sensu-on-rhel-centos/#sensu-enterprise)

To install Uchiwa, please visit the [Uchiwa installation guide][uchiwa-install].

_NOTE: as mentioned above &ndash; installation and use of a dashboard is not
required for operating Sensu Core or Sensu Enterprise._

[sensu-enterprise]:       https://sensu.io/products/enterprise
[core-api]:               ../../api/overview
[enterprise-api]:         /sensu-enterprise/latest/api/
[uchiwa-install]:         /uchiwa/latest/getting-started/installation
