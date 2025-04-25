# Kubernetes-Autoscaling-fundamentals

<!-- ## Manual HPA

    ```bash
    k scale deployment/(dep-name) --replica=5 
    ```

## Manual VPA

    ```bash
    k scale deployment/(dep-name) --replica=5 
    ``` -->

## Horizontal Pod Autoscaling

HorizontalPodAutoscaler automatically updates a workload resource (such as a Deployment or StatefulSet), with the aim of automatically scaling the workload to match demand.

Horizontal scaling means that the response to increased load is to deploy more Pods. This is different from vertical scaling, which for Kubernetes would mean assigning more resources (for example: memory or CPU) to the Pods that are already running for the workload.

**Notes** :
- Horizontal pod autoscaling does not apply to objects that can't be scaled (for example: a **DaemonSet**.)
- The `Horizontal pod autoscaling controller`, running within the Kubernetes control plane

- The default interval is `15 seconds`  and it's set by `--horizontal-pod-autoscaler-sync-period` in `kube-controller-manager `

- The controller manager finds the target resource defined by the scaleTargetRef, then selects the pods based on the target resource's .spec.selector labels, and obtains the metrics from either `the resource metrics API` (for per-pod resource metrics), or `the custom metrics API (for all other metrics)`.




kubectl events hpa nginx-deployment | grep -i "ScalingReplicaSet"


```yaml
# controlplane ~ ➜  cat deployment.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-prometheus-app
  labels:
    app: flask-prometheus-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-prometheus-app
  template:
    metadata:
      labels:
        app: flask-prometheus-app
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/path: '/metrics'
        prometheus.io/port: '8080'
    spec:
      containers:
      - name: flask-prometheus-app
        image: kodekloud/flask-prometheus-app:2
        ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flask-prometheus-app-service
  labels:
    app: flask-prometheus-app
spec:
  selector:
    app: flask-prometheus-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-prometheus-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-prometheus-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100m"
```

```bash
kubectl run -it --rm --restart=Never curl-test --image=radial/busyboxplus:curl -- sh
```

```bash
kubectl run curlpod --image=alpine:latest --restart=Never -- sh -c "apk add --no-cache curl && curl -s http://flask-prometheus-app-service/metrics"

kubectl logs curlpod -f
```

#### Install the Prometheus Community Helm Chart in the prometheus namespace.

###### Add the Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

###### Update the Helm Repositories

```bash
helm repo update
```

##### Install Prometheus Using the Helm Chart

```bash
helm install prometheus prometheus-community/prometheus --namespace prometheus --create-namespace --version 26.1.0
```


#### Execute the following command to generate traffic to the flask-prometheus-app-service:

```bash
kubectl run curl-pod --image=alpine -- /bin/sh -c "apk add --no-cache curl && sleep 3600"
kubectl exec -it curl-pod -- /bin/sh
timeout 10s sh -c 'while true; do curl -s http://flask-prometheus-app-service > /dev/null; done'
```

##### Install Prometheus Adapter Using Helm under prometheus namespace

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter -n prometheus
```

To verify if the Prometheus Adapter was successfully installed, run the following command:
```bash
 kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```


### Configure Prometheus Adapter to Expose Custom Metrics



Refer to [prometheus-adapter-values-active-session.yaml](https://github.com/dipinthomas/k8s-autoscaling/blob/master/020-025/prometheus-adapter-values-active-session.yaml)
--- 

To update the Prometheus Adapter in the cluster, utilize the configuration file located at /root/prometheus-adapter-values-active-session.yaml. Execute the following command:

```bash
helm upgrade prometheus-adapter prometheus-community/prometheus-adapter -n prometheus --version 4.11.0 -f /root/prometheus-adapter-values-active-session.yaml
```

Example :

```yaml

# controlplane ~ ➜  kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second" | jq .

{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "flask-prometheus-app-b97b9f9df-lc9ws",
        "apiVersion": "/v1"
      },
      "metricName": "http_requests_per_second",
      "timestamp": "2025-04-24T17:41:23Z",
      "value": "0",
      "selector": null
    }
  ]
}

# controlplane ~ ➜  cat hpa.yml 
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-prometheus-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-prometheus-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100m"
controlplane ~ ➜  k create -f hpa.yml 
horizontalpodautoscaler.autoscaling/flask-prometheus-app-hpa created

# controlplane ~ ➜  
```


Set up RabbitMQ within your Kubernetes cluster using Helm with the following command:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

```bash
helm install rabbitmq bitnami/rabbitmq \
  --set metrics.enabled=true \
  --set metrics.serviceMonitor.enabled=true \
  --set metrics.serviceMonitor.type=detailed \
  --set metrics.serviceMonitor.additionalLabels.release=prometheus \
  --set metrics.serviceMonitor.namespace=default \
  --set metrics.prometheus.export=true \
  --set metrics.prometheus.port=9419 \
  --set service.prometheusPort=9419 \
  --set auth.username=admin \
  --set auth.password=adminpassword \
  --set auth.erlangCookie=secretcookie \
  --set service.headless.enabled=true
```


# VPA

## Installation

```bash
  git clone https://github.com/kubernetes/autoscaler.git
     cd autoscaler/vertical-pod-autoscaler
        ./hack/vpa-up.sh
```

#### Example

```yaml
# controlplane ~ ➜  cat flask-app.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image:  kodekloud/flask-session-app:1 
        ports:
        - name: http
          containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
  labels:
    app: flask-app
spec:
  selector:
    app: flask-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
---
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: flask-app
spec:
  # recommenders field can be unset when using the default recommender.
  # When using an alternative recommender, the alternative recommender's name
  # can be specified as the following in a list.
  # recommenders: 
  #   - name: 'alternative'
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: flask-app
  updatePolicy:
    updateMode: "Recreate"
    evictionRequirements:
      - resources: ["cpu", "memory"]
        changeRequirement: TargetHigherThanRequests
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 100Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
# controlplane ~ ➜  
```

**Notes** : to check the logs you have three vpa pods 

vpa-admission-controller-7f6bf78fc5-ljrzx   1/1     Running   0          3m11s
vpa-recommender-f47cd47bb-vstkw             1/1     Running   0          3m12s
vpa-updater-85f7d79d77-ntbp6                1/1     Running   0          3m12s

---

Flask application is running with only 1 replica pod.
The Vertical Pod Autoscaler (VPA) needs to evict (remove) the existing pod to create a new one with updated resource settings.
Kubernetes has a safety feature that prevents removing the last pod of a deployment to avoid service downtime.
When you have only 1 replica and VPA tries to evict it, Kubernetes blocks this action with the error message: "too few replicas".
VPA wants to optimize your pod's resources but cannot because Kubernetes is protecting your service availability.
As a result, VPA cannot apply its resource recommendations, and application cannot benefit from automatic resource optimization.


```yaml
status:
  conditions:
    - lastTransitionTime: "2025-01-15T08:10:03Z"
      status: "True"
      type: RecommendationProvided

recommendation:
  containerRecommendations:
    - containerName: flask-app
      memory:
        lowerBound: # this is the min request of mem by the vpa
            memeory: 262144k
        target:     # the idel request
            memeory: 262144k
        uncappedTarget: # the recommendation by the vpa before looking for the min and max value inside the config (this is what i recommend without your constarins)
            memeory: 262144k
        upperBound:
            memeory: 1000Mi
```