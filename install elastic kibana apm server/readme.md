# Install elastic search stack on openshift

1. create new name space, and switch to it:
```
oc create ns elastic
oc project elastic
```
2. create a new service account:
```
oc create sa elastic-getapp
```
3. add policy to this new service account:
```
oc adm policy add-scc-to-user restricted -z elastic-getapp
```
4. apply this file:

```
oc apply -f elastic.yaml
```
`elastic.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apm-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apm-server
  template:
    metadata:
      labels:
        app: apm-server
    spec:
      serviceAccountName: elastic-getapp
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
      containers:
        - name: apm-server
          image: docker.elastic.co/apm/apm-server:7.17.18
          ports:
            - containerPort: 8200
          readinessProbe:
            httpGet:
              path: /
              port: 8200
            initialDelaySeconds: 10
            periodSeconds: 10
          command: ["/bin/sh", "-c"]
          args:
            - |
              apm-server -e \
                -E apm-server.rum.enabled=true \
                -E setup.kibana.host=kibana:5601 \
                -E setup.template.settings.index.number_of_replicas=0 \
                -E apm-server.kibana.enabled=true \
                -E apm-server.kibana.host=kibana:5601 \
                -E output.elasticsearch.hosts=["elasticsearch:9200"]
---
apiVersion: v1
kind: Service
metadata:
  name: apm-server
spec:
  selector:
    app: apm-server
  ports:
    - protocol: TCP
      port: 8200
      targetPort: 8200
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:7.17.18
          ports:
            - containerPort: 9200
          readinessProbe:
            httpGet:
              path: /
              port: 9200
            initialDelaySeconds: 30
            periodSeconds: 10
          env:
            - name: discovery.type
              value: "single-node"
            - name: cluster.name
              value: "docker-cluster"
            - name: bootstrap.memory_lock
              value: "true"
            - name: cluster.routing.allocation.disk.threshold_enabled
              value: "false"
            - name: ES_JAVA_OPTS
              value: "-XX:UseAVX=2 -Xms1g -Xmx1g"
          resources:
            limits:
              memory: "2Gi"
      #     volumeMounts:
      #       - name: esdata
      #         mountPath: /usr/share/elasticsearch/data
      # volumes:
      #   - name: esdata
      #     persistentVolumeClaim:
      #       claimName: elastic-getapp
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
    - protocol: TCP
      port: 9200
      targetPort: 9200
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      serviceAccountName: elastic-getapp
      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:7.17.18
          ports:
            - containerPort: 5601
          readinessProbe:
            httpGet:
              path: /
              port: 5601
            initialDelaySeconds: 30
            periodSeconds: 10
          env:
            - name: ELASTICSEARCH_URL
              value: "http://elasticsearch:9200"
            - name: ELASTICSEARCH_HOSTS
              value: "http://elasticsearch:9200"
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
spec:
  selector:
    app: kibana
  ports:
    - protocol: TCP
      port: 5601
      targetPort: 5601
```