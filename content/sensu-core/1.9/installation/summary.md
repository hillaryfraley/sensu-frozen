---
title: "Installation Guide Summary"
linkTitle: "Summary"
description: "Read this page to learn what to do next after you successfully install and configure the Sensu services, one or more Sensu clients, and a Sensu dashboard."
weight: 13
product: "Sensu Core"
version: "1.9"
next: ../upgrading
previous: ../install-a-dashboard
menu:
  sensu-core-1.9:
    parent: installation
---

Congratulations - you have successfully installed and configured the Sensu
services, one or more Sensu clients, and a Sensu dashboard! With this basic
implementation in place, you have the core Sensu platform available for
development, testing and evaluation purposes. What's next?

## Next steps

The installation guide provided manual, step-by-step instructions to help the
user (that's you) learn how Sensu is installed and configured. It would not be
practical to manually install Sensu on every machine you wish to monitor. It was
for this reason that Sensu was designed to be deployed by a configuration
management tool, such as [Puppet][1] or [Chef][2]; see the official [Puppet
module][3] and [Chef cookbook][4] for more information about how to automate
your Sensu deployment.

Depending on your business requirements you may be interested in [learning more
about how Sensu Works][5], or in preparing Sensu for production.

### Build a Comprehensive Telemetry Solution

Conceptually, building out a monitoring solution using Sensu requires one or
more [Sensu servers][6], and automating the installation of Sensu
clients on all of your machines. With this core platform in place, you'll also
need to define [monitoring checks][7] for all of the services in your
application stack (e.g. proxy services, web services, database services, etc).
Additionally, it may be prudent to collect and analyze metrics for the same
services, by creating [metric collection][8] and [metric analysis][9] checks.
Finally, no comprehensive telemetry solution would be complete without
configuring notification routing groups (i.e. who gets notified for what?) and
defining alerting and escalation policies via [Sensu Handlers][10].

### Deploying Sensu into Production Environments

- [Deploying Sensu using configuration management tools][14]
- [Securing Sensu][12]
- [Scaling Sensu][13]

[1]:  http://puppet.com
[2]:  http://www.chef.io
[3]:  https://github.com/sensu/sensu-puppet
[4]:  https://github.com/sensu/sensu-chef
[5]:  ../../guides/overview
[6]:  #scaling-sensu
[7]:  ../../guides/intro-to-checks
[8]:  ../../guides/intro-to-checks#create-a-metric-collection-check
[9]:  ../../guides/intro-to-checks#create-a-metric-analysis-check
[10]: ../../guides/intro-to-handlers
[11]: https://helpdesk.sensuapp.com
[12]: ../../guides/securing-sensu
[13]: ../../guides/scaling-overview
[14]: ../configuration-management
