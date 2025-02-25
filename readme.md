# Install KIC with KGO

## create k8s cluster

e.g. `
kind create cluster
`

use cloud-provider-kind for loadbalancer configuration and leave it running (or use a different tool/port-forwarding)

```
sudo cloud-provider-kind
```

## Install KGO

install GW API CRDS

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
# kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```


```
helm repo add kong https://charts.konghq.com
helm repo update kong

helm upgrade --install kgo kong/gateway-operator -n kong-system --create-namespace  \
  --set image.tag=1.4
```

## Install Tenant resources

```
kubectl apply -f gateway-api/tenant1.yaml -f gateway-api/tenant2.yaml
```

## Export LB IPs

```
export LB1_IP=$(kubectl get svc --namespace t1-gw $(kubectl get svc --no-headers -o custom-columns=":metadata.name" -n t1-gw | grep '^dataplane-ingress-') -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export LB2_IP=$(kubectl get svc --namespace t2-gw $(kubectl get svc --no-headers -o custom-columns=":metadata.name" -n t2-gw | grep '^dataplane-ingress-') -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

```

## Test Routes

Curl test routes.

```
#Test tenant 1 route
curl -i ${LB1_IP}/httpbin/get


#Test tenant 2 route
curl -i ${LB2_IP}/httpbin/get
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

## Test Upgrade and different versions

The current situation is that both tenants run a different version of KIC and of the Gateway itself. 
T1: GW 3.4.0, KIC 3.3.0
T2: GW 3.9.0, KIC 3.4.1

Those version are in the range of supported version by the KGO, which are described [here](https://docs.konghq.com/gateway-operator/1.4.x/reference/version-compatibility/).

Run 
```
kubectl apply -f gateway-api/tenant1-upgrade-dp.yaml
```
 to upgrade the T1 GW to version 3.8.x which is the latest version that the KIC 3.3.0 supports as of the compatibility matrix [here](https://docs.konghq.com/kubernetes-ingress-controller/latest/reference/version-compatibility/#kong).

## Apply global plugins

For applying global plugins currently an IngressClass is still required. This is created in the tenant resources already, currently per tenant. It can be debated if there is value in sharing an ingress class to have global plugins be able to be shared across tenants.

Install the global plugins

```
kubectl apply -f gateway-api/tenant1-global-plugin.yaml -f gateway-api/tenant2-global-plugin.yaml 
```

The example configures file-log and correlation-id plugin for both tenants. Name collision needs to be prevented here, which is a drawback.

Test with the example curls listed above and see the different correlationId headers `Tenant1-Kong-Request-ID` and `Tenant2-Kong-Request-ID`. See the dataplane logs to view the requests being logged to stdout.

```
curl  -s ${LB1_IP}/httpbin/get | grep -i Kong-Request-ID
curl  -s ${LB2_IP}/httpbin/get | grep -i Kong-Request-ID
```


# Considerations 

* Gateway.spec.infrastructure --> currently not implemented by KGO/KIC
* different version of KIC in one cluster --> minor and patch version on upgrading (3 months) --> this works
* 3.10 in compat table? 
  *  next KIC version will be compatible with 3.4 and 3.10, the plan is to hold that dual LTS compatibility in the future, it was not done for 2.8 because of major version change
  * upgrading of DPs can be done from each version to another e.g. 3.4 to 3.7 directly even if not  LTS since it is a dbless setup
  * KGO does not have LTS plans as of now
  * Latest KGO will support all actively supported KIC versions in the latest major version (including LTS)
* only local and redis strategy for RLA/Caching in hybrid mode --> this is no change since kond hybrid DPs do not have database conections
* per route plugin added to demo
* 