# Install KIC with KGO

## create k8s cluster

e.g. `
kind create cluster
`

## Install KGO

install GW API CRDS

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```


```
helm repo add kong https://charts.konghq.com
helm repo update kong

helm upgrade --install kgo kong/gateway-operator -n kong-system --create-namespace  \
  --set image.tag=1.4
```

## Install Tenant resources

```
kubectl apply -f gateway-api/tenant1.yaml
kubectl apply -f gateway-api/tenant2.yaml
```

## Test Routes

use cloud-provider-kind for loadbalancer configuration and leave it running (or use a different tool/port-forwarding)

```
sudo cloud-provider-kind
```

Get LB IPs and curl test routes.

```
#Test tenant 1 route

export LB1_IP=$(kubectl get svc --namespace t1-gw $(kubectl get svc --no-headers -o custom-columns=":metadata.name" -n t1-gw | grep '^dataplane-ingress-') -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl ${LB1_IP}/t1-httpbin/get


#Test tenant 2 route

export LB2_IP=$(kubectl get svc --namespace t2-gw $(kubectl get svc --no-headers -o custom-columns=":metadata.name" -n t2-gw | grep '^dataplane-ingress-') -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl ${LB2_IP}/t2-httpbin/get
```

## Test to add a route to GW1 from a non-allowed namespace

* it should be enforced by k8s policies which namespaces can carry the tx-gw-access label to enforce security
* this creates a non-allowed route for tenant3 trying to attach it to tenant 1 gateway

```
kubectl apply -f gateway-api/non-allowed-route.yaml

curl ${LB1_IP}/t3-httpbin/get
# This should fail! with "no Route matched" error

# check the status of the Route and look for "NotAllowedByListener"
kubectl -n t3-service-1  get httproute proxy-from-k8s-to-httpbin -o yaml | yq .status
```


# Considerations 

* Gateway.spec.infrastructure --> maybe this is better used for Gateway specific configuration since it erases the need for different gateway classes
* global CORS setting in this setup
* different version of KIC in one cluster --> minor and patch version on upgrading (3 months)
* 