create curilimage pod and test service with curl:
k exec -it -n hnp test -- curl 10.98.55.237:8080 -v
k exec -it -n hnp test -- curl nginx-srv:8080 -v   (use coredns)

k delete pod -n hnp test --force (if need) 


-------------------------------------------------------------------------

if we have another dns server in network, we should add to corednds
k get cm -n kube-system coredns -o yaml


------------------------------------------------------------------------