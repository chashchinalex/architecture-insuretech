1. Подготовка окружения
```
minikube start --driver=docker
kubectl cluster-info
kubectl get nodes
```

2. Установка Prometheus Stack
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

3. Настройка приложения insuretech и экспорта метрик
```
kubectl apply -f Task2/deployment.yaml
kubectl apply -f Task2/service.yaml
kubectl apply -f Task2/servicemonitor.yaml
kubectl get pods -l app=insuretech
kubectl get svc insuretech
kubectl get servicemonitor insuretech -o yaml
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

4. Настройка Prometheus Adapter
```yaml
prometheus:
  url: http://kube-prometheus-stack-prometheus.monitoring.svc
  port: 9090
  scheme: http
  insecureSkipVerify: true

rules:
  default: false
  resource: []
  custom:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        as: "rps_per_pod"
      metricsQuery: |
        sum by (namespace, pod) (
          rate(http_requests_total{<<.LabelMatchers>>}[1m])
        )

metricsRelistInterval: 30s
logLevel: 6
```
```
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring -f Task2/prometheus-adapter.values.yaml
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .resources
```

5. Генерация нагрузки
```
kubectl port-forward svc/insuretech 8080:8080 >/dev/null 2>&1 &
locust -f Task2/locustfile.py --headless -u 100 -r 20 -t 3m --host http://localhost:8080
```

6. Проверка метрики RPS
```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/rps_per_pod?labelSelector=app%3Dinsuretech" | jq .
```

7. Настройка HPA по метрике rps_per_pod
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: insuretech-rps-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: insuretech
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: rps_per_pod
        target:
          type: AverageValue
          averageValue: 5
```
```
kubectl apply -f Task2/hpa.rps.yaml
kubectl get hpa insuretech-rps-hpa -w
```

8. Проверка масштабирования
```
kubectl get hpa insuretech-rps-hpa -w
kubectl get deploy insuretech
kubectl get pods -l app=insuretech -o wide
```
