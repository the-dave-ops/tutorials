# Adding Custom Targets to OpenShift Monitoring build-in Prometheus

Managing additional targets in the built-in Prometheus instance in OpenShift can be challenging. Trying to install an additinonal prometheus instance and deleting the existing Prometheus instance is not recommended, as it may impact the overall health of the cluster. 
Instead, an alternative approach involves deploying a separate Prometheus instance tailored to your specific needs, along with the existing one, and usint both of them for your needs.
the main idia is, that the 2nd instance of prometheus won't contain the `node-exporter` as a target. the `node-exporter` is allready installed by default in the `opensift-monitoring` project.

### Steps to Deploy a Separate Prometheus Instance:

1. **Install Additional Prometheus Instance:** Utilize the Bitnami Helm chart for Prometheus to deploy a separate instance. The Bitnami chart provides a reliable and customizable solution for deploying Prometheus in your OpenShift cluster.

```
helm install my-release oci://registry-1.docker.io/bitnamicharts/prometheus
```
this will install only Prometheus itself, without node-exporter.
(you can see this chart here: https://github.com/bitnami/charts/tree/main/bitnami/prometheus/#installing-the-chart).

2. Customize Prometheus Configuration: After installation, edit the ConfigMap `prometheus-server` to include your custom targets. This allows you to define specific scraping targets tailored to your application's metrics endpoints.

```
oc edit configmap prometheus-server -n <your_namespace>
```

Add your custom targets under the scrape_configs section of the ConfigMap.

here is a example of the code with `uptime-kuma` as a target:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: prometheus-server
  namespace: prometheus
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: prometheus
    app.kubernetes.io/version: 2.50.1
    helm.sh/chart: prometheus-0.12.1
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: prometheus
data:
  prometheus.yaml: |
    global:
      external_labels:
        monitor: prometheus
    scrape_configs:
      - job_name: 'uptime'
        scrape_interval: 30s
        scheme: http
        metrics_path: '/metrics'
        static_configs:
          - targets: ['my-uptime-kuma.getapp-tools.svc.cluster.local:3001']
        basic_auth: # Only needed if authentication is enabled (default) 
          username: david
          password: newstart
          ...
          ...
```
3. restart the deployment, so the changes will take affect. 
4. **add a new Grafana Data Source:** In Grafana, add the newly deployed Prometheus instance as a new data source. This enables you to visualize and analyze metrics collected by the custom Prometheus instance alongside the default OpenShift monitoring metrics.

![data-source-grafana.png](/data-source-grafana.png)

> the main idia of all of this is:
for showing **server hardware matrics** - use the build in prometheus as data source. but for **custom targes** (for examle: `uptime-kuma`, `k6`) use the new prometheus as a data source.
{.is-info}

> adding the build in prometheus as a datasorce involves using authentication method, and is not so easy. see here how to set it up.
https://wiki.linnovate.net/en/openshift/grafana-and-prometheus-openshift-logging
{.is-info}
