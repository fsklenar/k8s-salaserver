# K8S metrics server

### Repository

```
https://github.com/kubernetes-sigs/metrics-server/
```

### Installation

add **--kubelet-insecure-tls** into [monitoring/values.yaml](monitoring/values.yaml) - disabling certificate validation for local server

```
kubectl apply -f monitoring/values.yaml
```

# Ingress installation INGRESS-NGINX

https://kubernetes.github.io/ingress-nginx/deploy/

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
