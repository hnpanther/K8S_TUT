create curilimage pod and test service with curl:
k exec -it -n hnp test -- curl 10.98.55.237:8080 -v
k exec -it -n hnp test -- curl nginx-srv:8080 -v   (use coredns)

k delete pod -n hnp test --force (if need) 


-------------------------------------------------------------------------

if we have another dns server in network, we should add to corednds
k get cm -n kube-system coredns -o yaml


------------------------------------------------------------------------

livness -> check if service(pod) not live, restart pod
readiness -> whene container is ready, then service select it
headless service -> connect to pods without a service between request and pod

---------------------------------------------------------------------------

downward api is like metadata but it can save data in file

-------------------------------------------------------------------------



control plane(master)
-etcd(key value sacalable db)
-api server(center of communication)
-scheduler -> which node run pod and so on
-controller manager

data planee(worker)
-container runtime
-kubelet -> communicate between control plane and data plane and manage log and connect to container runtime
-kube-proxy -> connection inside data plane and outside data plane and outside cluster and so on


and we have some add-on like kuber dns server, dashboard, ingress controller, container network interface cni like calico


k get componentstatuses -> show status of k8s component
k get --raw='/readyz?verbose' -> more details about component

list of kubernetes component:
k get pod -n kube-system
k get pod -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system


--------------

etcd -> distributed key value database and save status of cluster
just api server connect to etcd.
everything apply or create with kubectl save in etcd
we can use etcd.ctl for backup and restore data from etcd
we should create job or schedule for backup

etcd use optimisitc concurency for updating data and use rafting mechanism for leader election

--------------
api server:
clinet(kubectl) -> http post request -> authentication plugin -> authorization plugin -> admission plugin -> resource validation
then send to etcd

-------------
scheduler -> decide pod run on which node
after create/apply saved to etcd, a mechnism watch etcd(api server) and then notify scheduler
scheduler connect to etcd via api server not direct
after that scheduler update pod manifest and say this pod run on which code
we have another watch mechanism and after deciding scheduler, kublet on destination node, run pod
-------------
controller -> controller send command to kubelet, every resource in k8s has a controller(Deployment controller and so on)
indeed controller check everything and every change and notify kubelet
---------------
kubelet -> everything in workers manage by kubelet
kubelet run resource in workers and talk with container runtime, manage logs and register node and so on
kublet exists on masters and workers
---------------
kube proxy -> located in workers(and masters) and manage networking things, connect pod to service and api server and connection between pods and direct connect to iptables and create rules, connect to cni(calico) and so on
-------------
client create deployment with kubectl, send to api server and save it to etc, then notify a deployment controller
and deployment controller talk to api server and say create a replica set for me, then notify replica set controller
the say to api server to create a pod, then api server notify scheduler
and scheduler choose a worker and assign that to worker
then api server with watch mechanism notify kubelet on specific worker
and kubelet send command to container runtime(containerd) to pull and run pod
see all of this mechanism:
k get events --watch

how network work(calico and kube-proxy)? kube-proxy assign ip to pod
calico with kube proxy create some virtual interface
every pod has a virtual interface(eth0 for example with an ip)
so every pod has virtual interfice in own and another virtual interface at outside of every pod(like veth123)
then every node has a bridge
internal vi connect to outside vi and outside vi connect to bridge and bridge connect ro real interface on os(like ens33)
and we have real network between real interface and it's should be no NAT because every pod should be routable
kube-proxy write some rule in iptables and pods can talk together and routing handled

kube-proxy -> create iptable rules,load balancing,routing
calico -> create pod netwrok and assign ip for pods, routing between nodes, netwrok policy

----------------------------------

kubelet,kube-proxy,kubeadm,containerd not image and pod, everything else is pod

---------------------------------

security:

we have multiple builtin group in k8s:
authenticated user, unauthenticated user, service account, namespace service account
each pod has one service account, if we don't create new service accouont, default account attach to pod

we have rbac plugin that check authenticaton and autoriation

create new service account:
k create serviceaccount -n hnp hnp
k get sa -n hnp

-----
role based control access or rbac
role and rolebinding -> access in namespace
clusterrole and clusterrolebinding -> role in whole cluster like node 
role and pod is many to many releationship

list of clusterrolebinding:
k get clusterrolebindings.rbac.authorization.k8s.io
if we have permissive binding is so dengreous and should be delete in production

without permissive first create role and role binding then test:
kubectl auth can-i list services -n hnp2 --as=system:serviceaccount:hnp2:hnp

or use PodTestWithServiceAccount:
k exec -it -n hnp2 test -- curl localhost:8001/api/v1/namespaces/hnp2/services

some resource like persistent volume is cluster base, for that we should use ClusterRole and ClusterRoleBinding
then test:
k exec -it -n hnp2 test -- curl localhost:8001/api/v1/persistentvolumes

role and rolebinding is namespace base
we can use global clusterrole and attach to multiple rolebinding and so on

another way(correct way rather than custom kube-proxy) to test after enter to pod:
k exec -it -n hnp2 test -- sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default.svc/api/v1/persistentvolumes