## Logging & Monitoring Setup for Docker Containers on a Virtual Machine

In order to effectively monitor Docker containers running on a virtual machine (VM), it is essential to implement Grafana as a robust data visualization tool. Data retrieval from multiple sources is integral to this process. Logs will be collected via Loki and Promtail, while metrics will be gathered through Prometheus.

To accomplish this, the following microservices need to be installed on the VM:

### Logging Components:
1. **Grafana** - visualizion the data.
2. **Loki** - index the logs
3. **Promtail** - collect the logs

### Monitoring Components:
4. **Prometheus** - gather the matrics together.
5. **Node Exporter** - gets the VM matrics
6. **cAdvisor** - gets the matrics of the CONTAINERS
7. **Uptime Kuma** - shows the uptime status of containers

Below are the necessary files required for the installation of this comprehensive stack:

# PART 1: installing the infrastruture
## Step 1: install Grafana + Loki + Promtail:
docker compose file `docker-compose.yaml`

```
version: "3"

services:
  
  loki:
    container_name: loki
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki-data-volume:/loki
    command: -config.file=/etc/loki/local-config.yaml


  grafana:
    container_name: grafana
    image: grafana/grafana:latest
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
          - name: Loki
            type: loki
            access: proxy
            url: http://loki:3100
        EOF
        /run.sh
    ports:
      - "3030:3000"
    depends_on:
      - loki

  promtail:
    container_name: promtail
    image: grafana/promtail:2.8.0
    volumes:
      - ./promtail-local-config.yaml:/etc/promtail/config.yaml:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yaml

networks:
  default:
    name: urbalurba-network

volumes:
  loki-data-volume:
```
promtial config file `promtail-local-config.yaml`:

```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: flog_scrape 
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
```
copy these 2 files to the same directory (be aware to name the promtial config file `promtail-local-config.yaml` ) and then run:
```
docker-compose up || docker compose up
```

## Step 2: install prometheus:
docker compose file:

```
version: '3.8'

volumes:
  prometheus-data: 
    driver: local

services:
  prometheus:
    image: prom/prometheus:v2.37.9
    container_name: prometheus
    ports:
      - 9090:9090
    command: "--config.file=/etc/prometheus/prometheus.yaml"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yaml:ro
      - prometheus-data:/prometheus
    restart: unless-stopped
    networks:
      - prometheus

  node_exporter:
    image: quay.io/prometheus/node-exporter:v1.5.0
    container_name: node_exporter
    command: "--path.rootfs=/host"
    pid: host
    restart: unless-stopped
    volumes:
      - /:/host:ro,rslave
    networks:
      - prometheus


  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0     
    container_name: cadvisor
    ports:
      - 8090:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    restart: unless-stopped
    networks:
      - prometheus

networks:
  prometheus:
    external: true
```
`prometheus.yml` file:

```
  #Example job for node_exporter
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']

  # Example job for cadvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'uptime'
    scrape_interval: 30s
    scheme: http
    metrics_path: '/metrics'
    static_configs:
      - targets: ['getapp-test.getapp.sh:3001']
    basic_auth: # Only needed if authentication is enabled (default) 
      username: <uptime kuma username>
      password: <uptime kuma password>
```


## Step 3: install uptime kuma

`docker-compose.yaml` file

```
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - ./data:/app/data
    ports:
      - 3001:3001
    restart: unless-stopped
```
# PART 2: configure grafana
## Step 1: add Loki as data source:
text

## Step 2: add Prometheus as data source:
text

## Step 3: add Loki as data source:
text

