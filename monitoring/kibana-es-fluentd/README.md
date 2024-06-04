## Docu
https://docs.dapr.io/operations/observability/logging/fluentd/


## Add the helm repo for Elastic Search
`
helm repo add elastic https://helm.elastic.co
helm repo update
``

## Create PersistenVolume for ElasticSearch
`
kubectl apply -f monitoring/kibana-es-fluentd/elastic-pv.yaml
`

## Install Elastic Search using Helm
`
helm install elasticsearch elastic/elasticsearch --version 8.5.1 -n monitoring --set replicas=1
`

NOTES:
1. Watch all cluster members come up.
`
kubectl get pods --namespace=monitoring -l app=elasticsearch-master -w
`
2. Retrieve elastic user's password.
`
kubectl get secrets --namespace=monitoring elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
`
3. Test cluster health using Helm test.
`
helm --namespace=monitoring test elasticsearch
`

## Install Kibana
`
helm install kibana elastic/kibana --version 8.5.1 -n monitoring
`

## Install Fluentd

`
kubectl apply -f monitoring/fluentd-es/fluentd-config-map.yaml
kubectl apply -f monitoring/fluentd-es/fluentd-dapr-with-rbac.yaml
`
