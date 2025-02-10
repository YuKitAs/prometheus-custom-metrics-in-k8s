# README

This demo shows how to set up the infrastructure to expose custom metrics from a Flask application to Kubernetes' Custom Metrics API, which can be used for Horizontal Pod Autoscaler (HPA) to perform auto-scaling.

![](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/img/components.png)

## Environment

* Minikube: v1.35.0
* Docker (optional, can be replaced by Minikube's built-in Docker daemon)
* Kubectl (optional, can be replaced by `minikube kubectl`)
* Helm (optional, used to install Prometheus and Prometheus Adapter in this demo)
* Python 3.12

## Walkthrough

1. Create a Flask app and collect the custom metric `http_requests_total`, which can be found at `localhost:5000/metrics`.


2. Start Minikube with
```console
$ minikube start [--driver=docker]
```

If Docker is installed, it should be used as the default driver.


3. Build a Docker image with Minikube's internal Docker:
```console
$ eval $(minikube docker-env)
$ docker build -t localhost/prometheus-custom-metrics-in-k8s .
```

Adding `localhost` here because the default namespace would be `docker.io/library`.

**Note**: if Minikube can't find the built image, try this workaround:
```console
$ docker image save -o prometheus-custom-metrics-in-k8s.tar localhost/prometheus-custom-metrics-in-k8s
$ minikube image load prometheus-custom-metrics-in-k8s.tar
```


4. Deploy the app to K8s with
```console
$ kubectl apply -f k8s/
```

[deployment.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/deployment.yaml) and [service.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/service.yaml) will be applied to create a Deployment and a Service in the `default` namespace.


5. Verify the app is running:
```console
$ kubectl port-forward svc/prometheus-custom-metrics-in-k8s 5000:80
$ curl localhost:5000/hello
```


6. Add `prometheus-community` to Helm repo:
```console
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
```


7. Install Prometheus to `monitoring` namespace with Helm:
```console
$ helm install prometheus prometheus-community/prometheus -n monitoring --create-namespace
```

Alternatively, `prometheus-community/kube-prometheus-stack` can be installed, but in this example not all the components are needed.


8. After Prometheus is installed, we can find the Service `prometheus-server`, it's exposing port 80 by default:
```console
$ kubectl -nmonitoring get svc prometheus-server
```


9. We need [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) to translate K8s API requests into PromQL queries 
for Prometheus, so that it can fetch specific metrics for Custom Metrics API. Install Prometheus Adapter into the `monitoring` namespace with helm:
```console
$ helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring --set prometheus.url=http://prometheus-server.monitoring.svc --set prometheus.port=80
```

It's important to set the correct url and port here. 
If you chose to install `kube-prometheus-stack`, the Prometheus Server will be managed by the Prometheus Operator, and these configs will be different, then you should check the Service `prometheus-kube-prometheus-prometheus` instead.


10. Add a job to the ConfigMap of Prometheus Server:
```console
$ kubectl -nmonitoring get cm prometheus-server -o yaml > k8s/prometheus-config.yaml
```

Add a new job under `scrape_configs`:

```yaml
- job_name: prometheus-custom-metrics-in-k8s
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
  metrics_path: '/metrics'
  static_configs:
  - targets: ['prometheus-custom-metrics-in-k8s.default.svc.cluster.local:80']
```

The `relabel_configs` are needed to add `namespace` and `pod` labels to the custom metric so that it should look like this
```
http_requests_total{instance="<pod-ip>:5000", job="prometheus-custom-metrics-in-k8s", namespace="default", pod="<pod-name>"}
```

Apply the config:
```console
$ kubectl -nmonitoring apply -f k8s/prometheus-config.yaml
```

You could also edit the ConfigMap directly with
```console
$ kubectl -nmonitoring edit cm prometheus-server  
```


11. Restart the `prometheus-server` pod with the new config.


12. Add a rule to the ConfigMap of Prometheus Adapter:
```console
$ kubectl -nmonitoring get cm prometheus-adapter -o yaml > k8s/prometheus-adapter-config.yaml
```

Add a new query under `rules` to fetch the `http_requests_per_second` metric from Prometheus:

```yaml
- seriesQuery: 'http_requests_total'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total"
    as: "${1}_per_second"
  metricsQuery: sum(rate(http_requests_total[5m])) by (namespace, pod)
```

The `metricsQuery` can be validated on Prometheus UI at `localhost:9090` via port forwarding:

```
$ kubectl -nmonitoring port-forward svc/prometheus-server 9090:80
```

If it shows the correct result, apply the config:
```console
$ kubectl -nmonitoring apply -f k8s/prometheus-adapter-config.yaml
```

You could also edit the ConfigMap directly with
```console
$ kubectl -nmonitoring edit cm prometheus-adapter  
```


13. Restart the `prometheus-adapter` pod with the new config.


14. Verify the metric can be retrieved by Custom Metrics API:
```console
$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

In the `resources` array we should find the metric `namespaces/http_requests_per_second`. 
If `resources` is empty, check any errors in the logs in prometheus-adapter, most likely the adapter can't connect to Prometheus due to config error.
The connection can be tested with
```console
$ kubectl run -it --rm --image=curlimages/curl --restart=Never test -- curl -v <prometheus-url>:<prometheus-port>/api/v1/status/config
```