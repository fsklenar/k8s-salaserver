# 1. K8S metrics server

### Repository

```
https://github.com/kubernetes-sigs/metrics-server/
```

### Installation

add ```--kubelet-insecure-tls``` into [monitoring/metrics-server.yaml](monitoring/metrics-server.yaml) - disabling certificate validation for local server

```
kubectl apply -f monitoring/metrics-server.yaml
```

# 2. Ingress installation INGRESS-NGINX

https://github.com/kubernetes/ingress-nginx

###  Static LAN IP
By default ingress is not visible in your LAN, only internally from master/worker nodes.
First you have to set Static IP for ingress

- choose one IP from your LAN CIDR (example 192.168.0.202)
- add this IP to master node (netplan in Ubuntu)

### Prepare values.yaml for ingress-nginx

 - fill-in your LAN IP into [ingress-nginx/values.yaml](ingress-nginx/values.yaml)

### Installing via Helm Repository

To install the chart with the release name ingress-nginx:

For NGINX:

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    export INGRESS_NGINX_VER=4.10.1
    helm upgrade ingress-nginx --version ${INGRESS_NGINX_VER} -i -f ingress-nginx/values.yaml --namespace ingress-nginx --create-namespace ingress-nginx/ingress-nginx

# 3. Prometheus stack - prometheus operator

https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

## Documentation
Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

## Installation
- Note:
  - Install prometheus-kps into the new namespace (otherwise `default` namespace will be used)
  - Persistent Volumes are created for persisting data of Prometheus and Grafana
  - change  `externalIPs:` in `monitoring/prometheus/values.yaml` accordingly to your network
  - **EDIT** file [monitoring/prometheus/alertmanager.yaml](monitoring/prometheus/alertmanager.yaml) - replace `<xxx>` with proper values
```
kubectl create namespace monitoring
kubectl apply --namespace monitoring -f monitoring/prometheus/pv/prom-pv.yaml
kubectl apply --namespace monitoring -f monitoring/prometheus/pv/grafana-pv.yaml
kubectl create secret generic alertmanager-prometheus-kps-kube-promet-alertmanager \
  -n monitoring --from-file=monitoring/prometheus/alertmanager.yaml
helm upgrade -i prometheus-kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring -f monitoring/prometheus/values.yaml
```

### - Grafana

port forward:
```
 export GRAFANA_POD="$(kubectl get pods --namespace monitoring-l "app.kubernetes.io/name=grafana" --output jsonpath="{.items[0].metadata.name}")"
 kubectl port-forward -n monitoring ${GRAFANA_POD} 3000:3000
```
default username: **admin**
default password: **prom-operator**

### - Prometheus

port forward:
```
 export PROM_POD="$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus" --output jsonpath="{.items[0].metadata.name}")"
 kubectl port-forward -n monitoring ${PROM_POD}  9090:9090
```

### - Fix Kubeproxy alert


The metrics bind address of kube-proxy is default to `127.0.0.1:10249` that prometheus instances cannot access to. You should expose metrics by changing `metricsBindAddress` field value to `0.0.0.0:10249` if you want to collect them.

Depending on the cluster, the relevant part `config.conf` will be in ConfigMap kube-system/kube-proxy or kube-system/kube-proxy-config. For example:

```
kubectl -n kube-system edit cm kube-proxy
```
```
apiVersion: v1
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    # ...
    # metricsBindAddress: 127.0.0.1:10249
    metricsBindAddress: 0.0.0.0:10249
    # ...
  kubeconfig.conf: |-
    # ...
kind: ConfigMap
metadata:
  labels:
    app: kube-proxy
  name: kube-proxy
  namespace: kube-system
```

### - Similarly fix **Kube scheduler**, **Kube controller** and **etcd** alerts
- SSH login into control-plane node
```
cd /etc/kubernets/manifest
```
- change ``` - --bind-address=127.0.0.1``` in `kube-scheduler.yaml` file to ``` - --bind-address=0.0.0.0```
- change ``` - --bind-address=127.0.0.1``` in `kube-controller-manager.yaml` file to ``` - --bind-address=0.0.0.0```
- change ``` - --bind-address=127.0.0.1``` in `etcd.yaml` file to ``` - --listen-metrics-urls=http://0.0.0.0:2381```

# 4. Wordpress DEV
Wordpress application with MySQL

### - Generate MySQL password
```
kubectl create secret generic mysql-pass --from-literal=password=secret_db_pass -n wordpress-dev
```

### - Create Wordpress + MySQL deployments
```
kubectl apply -f wordpress-dev -n wordpress-dev
```

# 5. ElasticSearch + Fluentd + Kibana
### Original documentation
https://docs.dapr.io/operations/observability/logging/fluentd/


### Add the helm repo for Elastic Search
```
helm repo add elastic https://helm.elastic.co
helm repo update
```

### Create PersistentVolume for ElasticSearch
```
kubectl apply -f monitoring/kibana-es-fluentd/elastic/elastic-pv.yaml
```

### Install Elasticsearch using Helm
```
helm install elasticsearch elastic/elasticsearch --version 8.5.1 -n monitoring -f monitoring/kibana-es-fluentd/elastic/elastic.yaml
```

NOTES:
1. Watch all cluster members come up.
```
kubectl get pods --namespace=monitoring -l app=elasticsearch-master -w
```
2. Retrieve `elastic` user's password.
```
kubectl get secrets --namespace=monitoring elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```
3. COPY secret to `kube-system` namespace - `Fluentd` will need it
```
kubectl patch secret -n monitoring elasticsearch-master-credentials --type=json -p='[{"op": "replace", "path": "/metadata/namespace", "value": "kube-system"}]' -o yaml --dry-run=client | kubectl apply -f -
```
4. Test cluster health using Helm test.
```
helm --namespace=monitoring test elasticsearch
```

### Install Kibana
```
helm install kibana elastic/kibana --version 8.5.1 -n monitoring -f monitoring/kibana-es-fluentd/kibana/kibana.yaml
```
Patch Kibana service - add `ExternalIPs` - change IP accordingly to your network
```
kubectl patch svc kibana-kibana -p '{"spec":{"externalIPs":["10.192.168.202"]}}' -n monitoring
```

- **Port forward:**
Browse Kibana on localhost
```
kubectl port-forward svc/kibana-kibana 5601 -n monitoring
```
Browse to http://localhost:5601

### Install Fluentd

```
kubectl apply -f monitoring/kibana-es-fluentd/fluentd/fluentd-config-map.yaml
kubectl apply -f monitoring/kibana-es-fluentd/fluentd/fluentd-with-rbac.yaml
```

