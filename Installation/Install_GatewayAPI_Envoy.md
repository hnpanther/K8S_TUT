

create demo app with namespace and service ... and check:
kubectl get pods -n demo
kubectl get svc  -n demo


first install envoy gateway and api gateway crd:
kubectl apply --server-side -f https://github.com/envoyproxy/gateway/releases/download/v1.6.0/install.yaml

check everything is ok:
kubectl get pods -n envoy-gateway-system
kubectl get svc  -n envoy-gateway-system
kubectl get crds | grep gateway


create gatewayclass
check:
kubectl get gatewayclass


create gateway
check:
kubectl get gateway -A
kubectl describe gateway demo-gw -n demo
if programmed is false is not problem. it can't get external ip via load balancer!!!!!!!!


install httproute
check:
kubectl get httproute -A
kubectl describe httproute http-echo-route -n demo

in below command attached rule must be one:
kubectl describe gateway demo-gw -n demo

run below command to find nodeport port(envoy create service per gateway):
kubectl get svc -n envoy-gateway-system   --selector=gateway.envoyproxy.io/owning-gateway-namespace=demo,gateway.envoyproxy.io/owning-gateway-name=demo-gw   -o wide

then use ip node with port and header demo.local for test in browser
in this scenaio we should create gateway per namespace

if we want one gateway for all namespace we should create gateway in this way:
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: edge
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      # if we want all hostname accepted
      # hostname: "*.local"
      allowedRoutes:
        namespaces:
          from: All

---------------------------------

