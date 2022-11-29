# Zabbix Exporter

This project is a fork of [original zabbix exporter](https://github.com/MyBook/zabbix-exporter) 
with advanced functionality: 
filtering was added for each zabbix metric with configuration-defined hosts and item names, which significantly 
improves execution time.

### Filtering logic
Filtering is supported via two fields of a zabbix 'item' instance:
- host
- name (item_name)

For each metric a list of available hosts (host names) may be specified. In this case Zabbix API search
parameters will be used to get metric items which only belong to the given host list. This
provides significant performance boost. Note: if no host is given ('enable_empty_hosts' must be set to True in configuration file),
then no host restriction is set, but hostname will remain unknown in prometheus output for all hosts not mentioned in the config.

After receiving items, they are manually filtered by item_names using the following strategy.
We define two types of item_name filters:
- metric-specific item_names ('global' ones), which are visible within the concrete metric
- host-specific item_names ('local' ones), which ere visible within the concrete host of concrete metric 
(so that two equal hosts in different metrics do not share local filters).

Both item_name filters are presented as a set of item_name regular expressions (note: only asterisk is supported!).

So, getting to the item_name filtering logic itself, we have three possible situations:
- both global and local (for the specific host, of course) filters are missing. In this case all items will pass (no filters applied)
- either global or local filters are given. Then item will pass if its item_name suits at least one of the given masks
- both local and global filters are present. Same as in previous case, but mask lists are merged

### Usage example
```shell
Usage: zabbix_exporter [OPTIONS]
  Zabbix metrics exporter for Prometheus

  Use config file to map zabbix metrics names/labels into prometheus. Config
  below transforms this:
      local.metric[uwsgi,workers,myapp,busy] = 8
      local.metric[uwsgi,workers,myapp,idle] = 6

  into familiar Prometheus gauges:
      uwsgi_workers{instance="host1",app="myapp",status="busy"} 8
      uwsgi_workers{instance="host1",app="myapp",status="idle"} 6

  YAML config example:
    parsing:
      explicit_metrics: true
      enable_timestamps: false
      enable_empty_hosts: true  # if true, will load metrics with empty or missing 'hosts' field without host restriction
    
    metrics:
      - key: 'local.metric[uwsgi,workers,*,*]'
        name: 'uwsgi_workers'
        type: summary
        labels:
          app: $1
          status: $2
        reject:
          - 'total'
        hosts:                      # list of hosts to load this metric from
          - name.of.host.1          # simple host
          - name: name.of.host.2    # complex host with additional item_name masks
            item_names: 
              - '*host.specific.item.name*'
        item_names:                 # only items with names fitting one of the given patterns will be exported
          - '*item.name.substr.1*'
          - '*item.name.substr.2*'
      - key: 'metric.with.minimal.settings'
        name: 'minimal_example'
        type: summary


Options:
  --config PATH               Path to exporter config
  --port INTEGER              Port to serve prometheus stats [default: 9224]
  --url TEXT                  HTTP URL for zabbix instance
  --login TEXT                Zabbix username
  --password TEXT             Zabbix password
  --verify-tls / --no-verify  Enable TLS cert verification [default: true]
  --timeout INTEGER           API read/connect timeout
  --verbose
  --help                      Show this message and exit.

```    

### Deploying with Docker
```shell
docker run -d --name zabbix_exporter -v /path/to/your/config.yml:/zabbix_exporter/zabbix_exporter.yml --env=ZABBIX_URL="https://zabbix.example.com/" --env="ZABBIX_LOGIN=username" --env="ZABBIX_PASSWORD=secret" mybook/zabbix-exporter
```