Use Grafana NodeGraph for graphing network devices interactions:

#LLDP is good, but what it will show in the event of link/device failere.

example of using nodeGraph from (OpenTelemetry Collector)[https://grafana.com/grafana/dashboards/15983-opentelemetry-collector/]


```
'# receivers\nlabel_replace(\n  label_join(\n    label_join(\n      sum(${metric:value}(\n        otelcol_receiver_accepted_spans${suffix}{job="$job"}[$__rate_interval])\n      ) by (receiver)\n      , "id", "-rcv-", "transport", "receiver"\n    )\n    , "title", "", "transport", "receiver"\n  )\n  , "icon", "arrow-to-right", "", ""\n)\n\n# dummy processor\nor\nlabel_replace(\n  label_replace(\n    label_replace(\n      (sum(rate(otelcol_process_uptime${suffix}{job="$job"}[$__rate_interval])))\n      , "id", "processor", "", ""\n    )\n    , "title", "Processor(s)", "", ""\n  )\n  , "icon", "arrow-random", "", ""\n)\n\n# exporters\nor\nlabel_replace(\n  label_join(\n    label_join(\n      sum(${metric:value}(\n        otelcol_exporter_sent_spans${suffix}{job="$job"}[$__rate_interval])\n      ) by (exporter)\n      , "id", "-exp-", "transport", "exporter"\n    )\n    , "title", "", "transport", "exporter"\n  )\n  , "icon", "arrow-from-right", "", ""\n)'
```
