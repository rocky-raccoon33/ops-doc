![](img/gp.png)

## DEPLOY

[> Prometheus](http://localhost:3000/)

[> Grafana](http://localhost:9090)

[> PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)

[> QUERYING PROMETHEUS](https://prometheus.io/docs/prometheus/latest/querying/basics/)

```yml
# prometheus config
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    scheme: https
    static_configs:
      - targets: ["ops-test01"]

  - job_name: "spring-actuator"
    metrics_path: "/prometheus"
    scheme: https
    scrape_interval: 5s
    static_configs:
      - targets: ["ops-test01"]
```