# How to install grafana on a VM with Docker-compose
the original code is taken from this page. it whas tested and works OK.
https://community.grafana.com/t/hello-world-test-for-loki-promtail-and-grafana/90762/3
### docker-compose.yml file:

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
      - "3000:3000"
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

===

### promtail-local-config.yaml file:
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

just copy these 2 files to the same directory, and run:
```
docker-compose up
```

the grafana UI will be available at `<machine-ip>:3000`
the default usrname is `admin`, and the password is `admin` as well

to reaset a password (if you get locked out), you can enter the running container, and then run:
```
grafana-cli admin reset-admin-password NEW_PASSWORD
```
