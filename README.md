# README

This demo shows how to set up the infrastructure to expose custom metrics from a Flask application to Kubernetes' Custom Metrics API, which can be used for Horizontal Pod Autoscaler (HPA) to perform auto-scaling.

![](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/img/components.png)

- [Environment](#environment)
- [Walkthrough](#walkthrough)
  - [Deploy application](#deploy-application)
  - [Set up Prometheus](#set-up-prometheus)
      - [Option 1: Install `prometheus`](#option-1-install-prometheus)
      - [Option 2: Install `kube-prometheus-stack`](#option-2-install-kube-prometheus-stack)
  - [Set up Prometheus Adapter](#set-up-prometheus-adapter)
  - [Auto-scaling](#auto-scaling)

## Environment

* Minikube v1.35.0
* Docker v27.4.1 (optional, can be replaced by Minikube's built-in Docker daemon)
* Kubectl v1.32.0 (optional, can be replaced by `minikube kubectl`)
* Helm v3.17.0 (optional, used to install Prometheus and Prometheus Adapter in this demo)
* Python 3.12

## Walkthrough

### Deploy application

1. Create a Flask app ([app.py](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/app.py)) 
that exposes a custom metric `http_requests_total` at `http://localhost:5000/metrics` for Prometheus to scrape.


2. Start Minikube with
    
    ```console
    $ minikube start [--driver=docker]
    ```
    
    If Docker is installed, it should be used as the default driver.
    
    Now `kubectl` will be configured to use the `minikube` cluster.


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
    $ kubectl apply -f k8s/deployment.yaml
    $ kubectl apply -f k8s/service.yaml
    ```
    
    [deployment.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/deployment.yaml) and [service.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/service.yaml) will be applied to create a Deployment and a Service in the `default` namespace.


5. Verify the app is running:

    ```console
    $ kubectl port-forward svc/prometheus-custom-metrics-in-k8s 5000:80
    $ curl localhost:5000/hello
    ```

### Set up Prometheus

6. Add `prometheus-community` to Helm repo:

    ```console
    $ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    $ helm repo update
    ```

#### Option 1: Install `prometheus`

Only install a standalone Prometheus instance with manual configuration.

7. Install `prometheus` to `monitoring` namespace with Helm
    
    ```console
    $ helm install prometheus prometheus-community/prometheus -n monitoring --create-namespace
    ```


8. Verify Prometheus Service

    After `prometheus` is installed, we can find the Service `prometheus-server`, it's exposing port 80 by default:

    ```console
    $ kubectl -nmonitoring get svc prometheus-server
    ```
   

9. Add a job to the ConfigMap of Prometheus Server:
   ```console
   $ kubectl -nmonitoring get cm prometheus-server -o yaml > k8s/prometheus.cm.yaml
   ```

   Add a new scrape job under `scrape_configs`:
   
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
   
   Apply the config [prometheus.cm.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/prometheus.cm.yaml):
   ```console
   $ kubectl -nmonitoring apply -f k8s/prometheus.cm.yaml
   ```
   
   You could also edit the ConfigMap directly with
   ```console
   $ kubectl -nmonitoring edit cm prometheus-server  
   ```


10. Restart the `prometheus-server` pod with the new config.


11. Verify Prometheus config via port forwarding:

      ```console
      $ kubectl -nmonitoring port-forward svc/prometheus-server 9090:80
      ```
   
      Check `localhost:9090/config` and `localhost:9090/targets`.
   
      Alternatively with Minikube at a random port:
   
      ```console
      $ minikube -nmonitoring service prometheus-server
      ```


12. Install Prometheus Adapter into the `monitoring` namespace with helm

       ```console
       $ helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring --set prometheus.url=http://prometheus-server.monitoring.svc --set prometheus.port=80
       ```
    
13. Jump to step 14.


#### Option 2: Install `kube-prometheus-stack`

7. Alternatively, `kube-prometheus-stack` can be installed with additional components (e.g. Grafana and Alertmanager) and automatic service discovery. Prometheus will be managed by the Prometheus Operator instead.

    ```console
    $ helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
    ```


8. Verify Prometheus Service
   
    After `kube-prometheus-stack` is installed, the Service is called `prometheus-kube-prometheus-prometheus` and it's exposing port 9090:

    ```console
    $ kubectl -nmonitoring get svc prometheus-kube-prometheus-prometheus
    ```


9. Since the configuration is managed by the Prometheus Operator, new scrape jobs have to be defined using ServiceMonitor or PodMonitor. Create a ServiceMonitor ([service-monitor.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/service-monitor.yaml)):

   ```yaml
   spec:
     namespaceSelector:
       matchNames:
         - default
     selector:
       matchLabels:
         app: prometheus-custom-metrics-in-k8s
     endpoints:
       - port: metrics
         path: /metrics
         interval: 10s
   ```
   
   **Note**: it needs to be allowed to discover services in the `default` namespace, the labels must match the service labels, and the port must match a named port in the service.


10. Deploy ServiceMonitor to `monitoring` namespace:

      ```console
      $ kubectl apply -f k8s/service-monitor.yaml
      ```


11. Make sure Prometheus can select ServiceMonitors from other namespaces by checking that the Prometheus CRD

      ```console
      $ kubectl -nmonitoring describe prometheus prometheus-kube-prometheus-prometheus
      ```
      
      contains
      
      ```yaml
      spec:
        serviceMonitorNamespaceSelector: {}  # Selects ServiceMonitors from all namespaces
        serviceMonitorSelector: {}  # Selects all ServiceMonitors
      ```


12. Verify Prometheus config via port forwarding:

      ```console
      $ kubectl -nmonitoring port-forward svc/prometheus-operated 9090
      ```
      
      Check `localhost:9090/config` and `localhost:9090/targets`.
      
      Or with Minikube at a random port:
      
      ```console
      $ minikube -nmonitoring service prometheus-kube-prometheus-prometheus
      ```


13. Install Prometheus Adapter into the `monitoring` namespace with helm

    ```console
    $ helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring --set prometheus.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc --set prometheus.port=9090
    ```


### Set up Prometheus Adapter

14. Add a rule to the ConfigMap of Prometheus Adapter:

      ```console
      $ kubectl -nmonitoring get cm prometheus-adapter -o yaml > k8s/prometheus-adapter.cm.yaml
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
        metricsQuery: sum(rate(http_requests_total[2m])) by (namespace, pod)
      ```
      
      The `metricsQuery` can be validated on Prometheus UI, if it shows the correct result, apply the config:
   
      ```console
      $ kubectl -nmonitoring apply -f k8s/prometheus-adapter.cm.yaml
      ```
      
      You could also edit the ConfigMap directly with
      ```console
      $ kubectl -nmonitoring edit cm prometheus-adapter
      ```
   
      More details about the config can be found in the [official doc](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md).


15. Restart the `prometheus-adapter` pod with the new config.


16. Verify the metric can be retrieved by Custom Metrics API:

      ```console
      $ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
      ```
      
      In the `resources` array you should find the metric `pods/http_requests_per_second`. 
      If `resources` is empty, check any errors in the logs in prometheus-adapter, most likely the adapter can't connect to Prometheus due to config error.
      The connection can be tested with
      ```console
      $ kubectl run -it --rm --image=curlimages/curl --restart=Never test -- curl -v <prometheus-url>:<prometheus-port>/api/v1/status/config
      ```

### Auto-scaling

17. Create a HorizontalPodAutoscaler that scales based on the custom metric like in [hpa.yaml](https://github.com/YuKitAs/prometheus-custom-metrics-in-k8s/blob/main/k8s/hpa.yaml):

      ```yaml
      metrics:
        - type: Pods
          pods:
            metric:
              name: http_requests_per_second
            target:
              type: AverageValue
              averageValue: 20
      ```

18. Deploy the HPA to the `default` namespace as well:

      ```console
      $ kubectl apply -f k8s/hpa.yaml
      ```

19. Run a load test to simulate more than 20 requests per second for 2 minutes. The HPA event log should show the following:

      ```
      $ kubectl describe hpa prometheus-custom-metrics-in-k8s
      
      Events:
      Normal  SuccessfulRescale  7m  horizontal-pod-autoscaler    New size: 2; reason: pods metric http_requests_per_second above target
      Normal  SuccessfulRescale  1m  horizontal-pod-autoscaler    New size: 1; reason: All metrics below target
      ```
      
      You can also monitor the pods with
      ```console
      $ kubectl get pod -w
      ```
