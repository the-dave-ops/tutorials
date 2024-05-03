# How to deploy promtail on openshift
Promtail is part of the loki-grafana stack.
It is the part of the stack that taks care of collecting the logs from all the pods in the cluster.
Its deployed as deamonset.
when deploying on openshift there will be some securety issues, so we need to preform these steps:

1. find the name of the service account name:

```
      serviceAccountName: loki-promtail
```
then add the scc polices to this existing service account:
```
oc adm policy add-scc-to-user hostmount-anyuid -z loki-promtail

```
this allows to promtail to run a `hostpath` volume, and to run as user 0.
see here: https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/troubleshooting/#openshift-support

2. you will need to change the securety conetxt as well. here is a example of a yaml file with the correct settings.
see lins 21-24 45-51
```
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: loki-promtail
spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: loki
      app.kubernetes.io/name: promtail
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: loki
        app.kubernetes.io/name: promtail
    spec:
      restartPolicy: Always
      serviceAccountName: loki-promtail
      schedulerName: default-scheduler
      enableServiceLinks: true
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0
      containers:
        - resources: {}
          readinessProbe:
            httpGet:
              path: /ready
              port: http-metrics
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          terminationMessagePath: /dev/termination-log
          name: promtail
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          securityContext:
            capabilities:
              drop:
                - ALL
            privileged: true
            runAsUser: 0
            runAsGroup: 0
          ports:
            - name: http-metrics
              containerPort: 3101
              protocol: TCP
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: run
              mountPath: /run/promtail
            - name: config
              mountPath: /etc/promtail
            - name: containers
              readOnly: true
              mountPath: /var/lib/docker/containers
            - name: pods
              readOnly: true
              mountPath: /var/log/pods
          terminationMessagePolicy: File
          image: 'docker.io/grafana/promtail:2.9.2'
          args:
            - '-config.file=/etc/promtail/promtail.yaml'
      serviceAccount: loki-promtail
      volumes:
        - name: config
          secret:
            secretName: loki-promtail
            defaultMode: 420
        - name: run
          hostPath:
            path: /run/promtail
            type: ''
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
            type: ''
        - name: pods
          hostPath:
            path: /var/log/pods
            type: ''
      dnsPolicy: ClusterFirst
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
  revisionHistoryLimit: 10
```