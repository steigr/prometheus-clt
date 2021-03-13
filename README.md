# Prometheus Stack for CLT talk

This repo contains all manifests for running a simple prometheus + grafana setup on an existing kubernetes cluster.

It will install a list of workloads:

```
kubectl apply -f prometheus-operator/
kubectl apply -f prometheus/
kubectl apply -f grafana/
```

## Adding own servers

To add a type of monitored server (can be more than one) create the following 3 items on the kubernetes system:

1. Service of the `ExternalName` pointing towards the destination server(s). Add labels to hint prometheus which data
   can be acquired on those servers (like `web-server: "true"`).
   
    __Example__:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: web
  namespace: default
  labels:
    app: web
    metric-router: true
    metric-httpd2: true
    metric-node: true
spec:
  type: ExternalName
  externalName: web.server
  ports:
    - name: metrics
      port: 9090
```
   
2. Create Endpoints - as Prometheus will only scrape services with endpoints, and the service is external (which does not yield endpoints at all)
   you have to manage them by yourself:

   __Example__:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: web
  namespace: default
subsets:
  - addresses:
      - ip: 10.211.55.17
      # Add additional web servers here
      # - ip: 08.15.08.15 
    ports:
      - name: metrics
        port: 9090
        protocol: TCP
```

3. Create ServiceMonitors - Prometheus' auto-discovery will pick them up, merge them with the endpoints to generate the list of to-be-scraped endpoints.

   __Example__:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: httpd2
  namespace: default
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      metric-httpd2: active
  endpoints:
    - port: metrics
      path: /metrics/httpd2
```

## Adding own dashboards

To add own dashboards you will have to add them as ConfigMap to the Kubernetes and update Grafana's deployment.

1. Create Dashboard Configmap - grafana will discover them automatically.

   __Example__:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-httpd2
  namespace: default
data:
  httpd2.json: |-
    {
    ...
    }
```

2. Mount the dashboard within Grafana.

   __Example__:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: default
  # ...
spec:
  # ...
  template:
     # ...
     spec:
      containers:
        - name: grafana
          # ...
          volumeMounts:
             - mountPath: /grafana-dashboard-definitions/0/httpd2
               name: grafana-dashboard-httpd2
               readOnly: false
            # ...
      volumes:
        - name: grafana-dashboard-httpd2
          configMap:
            name: grafana-dashboard-httpd2
          # ...
```

## TBD

### Agent inquiry

Implement agent discovery and registration. The agent on the to-be-scraped target servers can (and once had) expose(d)
a list of running exporters. Therefore a 3rd service (besides Prometheus and Grafana) will check the ServiceMonitors.
If they are pointing towards a Service of type External without Endpoints this is a candidate for a legacy server.

This service registers the server(s) behind the Services externalNames as endpoint if on the to-be-scraped port a
metrics-inquiry (GET /.well-known/metrics) is successful and the ServiceMonitors endpoint path is found in the exposed
metrics on the agent.

### Dashboards auto-updater

Add dashboards on the fly. There is another service for Grafana to pull in new dashboards from Kubernetes ConfigMap
automatically.
