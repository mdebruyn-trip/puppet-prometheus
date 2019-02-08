# puppet-prometheus

[![Build Status](https://travis-ci.org/voxpupuli/puppet-prometheus.svg?branch=master)](https://travis-ci.org/voxpupuli/puppet-prometheus)
[![Puppet Forge](https://img.shields.io/puppetforge/v/puppet/prometheus.svg)](https://forge.puppetlabs.com/puppet/prometheus)
[![Puppet Forge - downloads](https://img.shields.io/puppetforge/dt/puppet/prometheus.svg)](https://forge.puppetlabs.com/puppet/prometheus)
[![Puppet Forge - endorsement](https://img.shields.io/puppetforge/e/puppet/prometheus.svg)](https://forge.puppetlabs.com/puppet/prometheus)
[![Puppet Forge - scores](https://img.shields.io/puppetforge/f/puppet/prometheus.svg)](https://forge.puppetlabs.com/puppet/prometheus)
[![Apache-2 License](https://img.shields.io/github/license/voxpupuli/puppet-prometheus.svg)](LICENSE)

## Table of Contents

* [Compatibility](#compatibility)
* [Background](#background)
* [Usage](#usage)
* [Example](#example)
* [Known Issues](#known-issues)
* [Development](#development)
* [Authors](#authors)

----

This module contains a Puppet [reports processor][rpc] that writes report
metrics in a format that is accepted by [Prometheus node_exporter Textfile
Collector][pnetc].

[rpc]:https://docs.puppet.com/puppet/latest/reference/reporting_about.html
[pnetc]:https://github.com/prometheus/node_exporter#textfile-collector

node_exporter >= 0.15.0
consul_exporter >= 0.3.0

This module supports below Prometheus architectures:
- x86_64/amd64
- i386
- armv71 (Tested on raspberry pi 3)

## How to

### Puppet setup

Include this module in your path, and create a file named `prometheus.yaml` in
your Puppet configuration directory.

* Installs the prometheus daemon, alertmanager or exporters(via url or package)
  * The package method was implemented, but currently there isn't any package for prometheus
* Optionally installs a user to run it under (per exporter)
* Installs a configuration file for prometheus daemon (/etc/prometheus/prometheus.yaml) or for alertmanager (/etc/prometheus/alert.rules)
* Manages the services via upstart, sysv, or systemd
* Optionally creates alert rules
* The following exporters are currently implemented: node_exporter, statsd_exporter, process_exporter, haproxy_exporter, mysqld_exporter, blackbox_exporter, consul_exporter, redis_exporter, varnish_exporter, graphite_exporter, postgres_exporter, collectd_exporter

Include `prometheus` in your Puppet reports configuration; enable pluginsync:

**Notice about breaking changes**

Version 5.0.0 and older of this module allowed you to deploy the prometheus server by doing a simple `include prometheus`.
We introduced a [new class layout](https://github.com/voxpupuli/puppet-prometheus/pull/194) in
version 6. By default, including the `prometheus` class won't deploy the server now.
You need to include the `prometheus::server` class for this (which has the same
parameters that `prometheus` had). An alternative approach is to set the
`manage_prometheus_server` parameter to true in the `prometheus` class. Background information about this change is described in the related [pull request](https://github.com/voxpupuli/puppet-prometheus/pull/187) and the [issue](https://github.com/voxpupuli/puppet-prometheus/issues/184).

To set up a prometheus daemon:
On the server (for prometheus version < 1.0.0):

```puppet
class { 'prometheus::server':
  global_config  => { 'scrape_interval'=> '15s', 'evaluation_interval'=> '15s', 'external_labels'=> { 'monitor'=>'master'}},
  rule_files     => [ "/etc/prometheus/alert.rules" ],
  scrape_configs => [
     { 'job_name'=> 'prometheus',
       'scrape_interval'=> '10s',
       'scrape_timeout' => '10s',
       'target_groups'  => [
        { 'targets'     => [ 'localhost:9090' ],
            'labels'    => { 'alias'=> 'Prometheus'}
         }
      ]
    }
  ]
}
```

Note: you can use a comma separated list of reports processors:

```puppet
class { 'prometheus::server':
    version => '1.0.0',
    scrape_configs => [ {'job_name'=>'prometheus','scrape_interval'=> '30s','scrape_timeout'=>'30s','static_configs'=> [{'targets'=>['localhost:9090'], 'labels'=> { 'alias'=>'Prometheus'}}]}],
    extra_options => '-alertmanager.url http://localhost:9093 -web.console.templates=/opt/prometheus-1.0.0.linux-amd64/consoles -web.console.libraries=/opt/prometheus-1.0.0.linux-amd64/console_libraries',
    localstorage => '/prometheus/prometheus',
}
```
On the server (for prometheus version >= 2.0.0):

```puppet
class { 'prometheus::server':
    version        => '2.4.3',
    alerts => { 'groups' => [{ 'name' => 'alert.rules', 'rules' => [{ 'alert' => 'InstanceDown', 'expr' => 'up == 0', 'for' => '5m', 'labels' => { 'severity' => 'page', }, 'annotations' => { 'summary' => 'Instance {{ $labels.instance }} down', 'description' => '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.' }}]}]},
    scrape_configs => [
      { 'job_name' => 'prometheus',
        'scrape_interval' => '10s',
        'scrape_timeout'  => '10s',
        'static_configs'  => [
        { 'targets' => [ 'localhost:9090' ],
          'labels'  => { 'alias' => 'Prometheus'}
       }
      ]
    }
  ]
}
```

or simply:
```puppet
include prometheus::server
```

To add alert rules, add the following to the class prometheus in case you are using prometheus < 2.0:
```puppet
    alerts => [{ 'name' => 'InstanceDown', 'condition' => 'up == 0', 'timeduration' => '5m', labels => [{ 'name' => 'severity', 'content' => 'page'}], 'annotations' => [{ 'name' => 'summary', content => 'Instance {{ $labels.instance }} down'}, {'name' => 'description', content => '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.' }]}]
```

### Sample

```
puppet_report_resources{name="Changed",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Failed",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Failed to restart",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Out of sync",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Restarted",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Scheduled",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Skipped",environment="production",host="node.example.com"} 0
puppet_report_resources{name="Total",environment="production",host="node.example.com"} 519
puppet_report_time{name="Acl",environment="production",host="node.example.com"} 3.8629975709999984
puppet_report_time{name="Anchor",environment="production",host="node.example.com"} 0.002442332
puppet_report_time{name="Augeas",environment="production",host="node.example.com"} 10.629003954
puppet_report_time{name="Concat file",environment="production",host="node.example.com"} 0.0026740609999999997
puppet_report_time{name="Concat fragment",environment="production",host="node.example.com"} 0.012010700000000003
puppet_report_time{name="Config retrieval",environment="production",host="node.example.com"} 20.471957786
puppet_report_time{name="Cron",environment="production",host="node.example.com"} 0.000874118
puppet_report_time{name="Exec",environment="production",host="node.example.com"} 0.4114313850000001
puppet_report_time{name="File",environment="production",host="node.example.com"} 0.32955574000000015
puppet_report_time{name="File line",environment="production",host="node.example.com"} 0.002702939
puppet_report_time{name="Filebucket",environment="production",host="node.example.com"} 0.0003994
puppet_report_time{name="Grafana datasource",environment="production",host="node.example.com"} 0.187452552
puppet_report_time{name="Group",environment="production",host="node.example.com"} 0.0031514940000000003
puppet_report_time{name="Mysql datadir",environment="production",host="node.example.com"} 0.000422795
puppet_report_time{name="Package",environment="production",host="node.example.com"} 1.670733222
puppet_report_time{name="Service",environment="production",host="node.example.com"} 0.8740041969999999
puppet_report_time{name="Total",environment="production",host="node.example.com"} 38.468031933999995
puppet_report_time{name="User",environment="production",host="node.example.com"} 0.005163427
puppet_report_time{name="Yumrepo",environment="production",host="node.example.com"} 0.0010542610000000001
puppet_report_changes{name="Total",environment="production",host="node.example.com"} 0
puppet_report_events{name="Failure",environment="production",host="node.example.com"} 0
puppet_report_events{name="Success",environment="production",host="node.example.com"} 0
puppet_report_events{name="Total",environment="production",host="node.example.com"} 0
puppet_report{environment="production",host="node.example.com"} 1477054915347
```

When using prometheus >= 2.0, we use the new yaml format (https://prometheus.io/docs/prometheus/2.0/migration/#recording-rules-and-alerts) configuration

```puppet
    alerts => { 'groups' => [{ 'name' => 'alert.rules', 'rules' => [{ 'alert' => 'InstanceDown', 'expr' => 'up == 0', 'for' => '5m', 'labels' => { 'severity' => 'page', }, 'annotations' => { 'summary' => 'Instance {{ $labels.instance }} down', 'description' => '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.' } }]}]},
```

```yaml
alerts:
  groups:
    - name: alert.rules
      rules:
      - alert: 'InstanceDown'
        expr: 'up == 0'
        for: '5m'
        labels:
          'severity': 'page'
        annotations:
          'summary': 'Instance {{ $labels.instance }} down'
          'description': '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
```

On the monitored nodes:

```puppet
   include prometheus::node_exporter
```

Special thanks to [Puppet, Inc](http://puppet.com) for Puppet, and its store
reports processor, to [EvenUp](https://letsevenup.com/) for their
[graphite](https://github.com/evenup/evenup-graphite_reporter) reports
processor, and to [Vox Pupuli](https://voxpupuli.org) to provide a platform
that allows us to develop of this module.

```puppet
class { 'prometheus::node_exporter':
    version => '0.12.0',
    collectors_disable => ['loadavg','mdadm' ],
    extra_options => '--collector.ntp.server ntp1.orange.intra',
}
```

For more information regarding class parameters please take a look at class docstring.

## Example

Real Prometheus >=2.0.0 setup example including alertmanager and slack_configs.

```puppet
include profiles::prometheus

class { 'prometheus':
  manage_prometheus_server => true,
  version => '2.0.0',
  alerts => { 'groups' => [{ 'name' => 'alert.rules', 'rules' => [{ 'alert' => 'InstanceDown', 'expr' => 'up == 0', 'for' => '5m', 'labels' => { 'severity' => 'page', }, 'annotations' => { 'summary' => 'Instance {{ $labels.instance }} down', 'description' => '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.' } }]}]},
  scrape_configs => [
    { 'job_name' => 'prometheus',
      'scrape_interval' => '10s',
      'scrape_timeout'  => '10s',
      'static_configs'  => [
         { 'targets' => [ 'localhost:9090' ],
           'labels'  => { 'alias'=> 'Prometheus'}
         }
      ]
    },
    { 'job_name' => 'node',
      'scrape_interval' => '5s',
      'scrape_timeout'  => '5s',
      'static_configs'  => [
         { 'targets' => [ 'nodexporter.domain.com:9100' ],
           'labels'  => { 'alias'=> 'Node'}
         }
      ]
    }
  ],
  alertmanagers_config => [{ 'static_configs' => [{'targets' => [ 'localhost:9093' ]}]}],
}
class { 'prometheus::alertmanager':
  version       => '0.13.0',
  route         => { 'group_by' => [ 'alertname', 'cluster', 'service' ], 'group_wait'=> '30s', 'group_interval'=> '5m', 'repeat_interval'=> '3h', 'receiver'=> 'slack' },
  receivers     => [ { 'name' => 'slack', 'slack_configs'=> [ { 'api_url'=> 'https://hooks.slack.com/services/ABCDEFG123456', 'channel' => '#channel', 'send_resolved' => true, 'username' => 'username'}] }]
}
```

the same in hiera

```yaml
prometheus::manage_prometheus_server: true
prometheus::version: '2.0.0'
prometheus::scrape_configs:
    - job_name: 'nodexporter'
      scrape_interval:  '10s'
      scrape_timeout: '10s'
      static_configs:
      - targets:
        - nodexporter.domain.com:9100
        labels:
          alias: 'nodexporter'
    - job_name: prometheus
      scrape_interval: 10s
      scrape_timeout: 10s
      static_configs:
      - targets:
        - localhost:9090
        labels:
          alias: Prometheus
prometheus::alerts:
  groups:
    - name: alert.rules
      rules:
      - alert: 'InstanceDown'
        expr: 'up == 0'
        for: '5m'
        labels:
          'severity': 'page'
        annotations:
          'summary': 'Instance {{ $labels.instance }} down'
          'description': '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.'
prometheus::alertmanagers_config:
  - static_configs:
    - targets:
      - localhost:9093

prometheus::alertmanager::version: '0.13.0'
prometheus::alertmanager::route:
  group_by:
  - alertname
  - cluster
  - service
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  receiver: slack
prometheus::alertmanager::receivers:
  - name: slack
    slack_configs:
    - api_url: https://hooks.slack.com/services/ABCDEFG123456
      channel: "#channel"
      send_resolved: true
      username: username
```
Test you commit with vagrant https://github.com/kalinux/vagrant-puppet-prometheus.git

## Known issues

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

This module has unit tests for CentOS/RHEL, Debian, Ubuntu and Archlinux. Acceptance tests are executed for CentOS, Debian and Ubuntu. Other operatingsystems may work but are untested.


## Development

This project contains tests for [rspec-puppet](http://rspec-puppet.com/).

Quickstart to run all linter and unit tests:

```bash
bundle install --path .vendor/ --without system_tests --without development --without release
bundle exec rake test
```

## Authors

puppet-prometheus is maintained by [Vox Pupuli](https://voxpupuli.org), it was written by [brutus333](https://github.com/brutus333/).
