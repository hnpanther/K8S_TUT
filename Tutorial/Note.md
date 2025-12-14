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

-------------------------------------------
Security -> HostSecurity
some kubernetes component like kube-proxy needs to change iptable host or connect to real interface of host
so we have something that help us to connect pod to real interface but it's a security issues
HostNetwork and HostPort exists for that
HostPID = true => our process in pod get a pid from host and HostIPC = true our pod use ipc namespace of host

using securityContext we can set HostPID and so on and we can set run as a user and so on
runAsNonRoot: true is so important

we have fsGroup supplementalGroups
supplementalGroups use for group add on default
fs group => all volume will be fs group for permissons
supplementalGroups => for extra group

for secure pod globally(secure all pod) we should use PodSecurityPolicy but is deprecated
and we should use PodSecurityAdmission or third parties(like kubewarden)
 with pod sevurity standard with 3 profile:
1-Priviledge => full access
2-Baseline => average
3-Restricted => best practice secure
before apply and create pod this plugin check and validate

for use in creation namespace:
apiVersion: v1
kind: Namespace
metadata:
  name: my-priviledge-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest



NetworkPolicy:
access pods together in same or different namespace
default: every pods can access every pods


-------------------------------------
Scheduler decide pod runs on which node:
ListRequestedPriority:
most free space
MostRequestedPriority:
low free space
so Scheduler watch request, not limit
in resource if we have 4000m cpu
and a pod request 200m and another pod request 1200m, and then two pod need more cpu, kuber continue 1/5 of remain for this pods

for java app is so important set xmx otherwise jvm use all ram(limit for jvm app not effect)

-- qos => quality of service:
1-best effort => lower qos
2-berse table
3-garanteed => higher qos
pod without any request and limit => best effort
pod with one container with unequal request and limit => berse table
pod with multiple container and one container has resource and another one doesn't have => berse table
pod with request=limit for all container(should set cpu and limit for all container in pod) => garanteed

oom => space of remain ram of container => set score base on oom for each container and if whenever need to delete a container, higher score delete first

for set limit in namespace on all limit: use LimitRange for default for all pods
k get limitrange

for set limitation on namespace we use ResourceQuota. for example a namespace should be 2g ram
if we apply ResourceQuota, every pod shoud have request and limit otherwise it's not run
so we'd better use LimitRange for Default for all pods

by k descrive resourcequotas  => we can see used and and hard resource


-----------------------------
Scheduling
we can use somethings to control scheduling on nodes
Taint => about node
Toleration => about pod
a pod if tolerate , taint of node, can deploy on this node
if we want to decide a pod run on a node or no we use taint and toleration
but if we want pod itself descide to run which node, it's about afinity and anti-afinity

for example if we describe master we can see:
Taints:             node-role.kubernetes.io/control-plane:NoSchedule => if a node has kubernters-role=control-plane, no pod can deploy on it or a pod should tolerate exatly this taint


node-role.kubernetes.io/control-plane:NoSchedule
key => node-role.kubernetes.io
value => control-plane
effect => NoSchedule  => just pod run this node if tolerate this
effect => PreferSchedule => if exists another need, schedule there
effect => NoExecute => running pod run on node 14, if we set node 14 NoExecute, pod transfer to another node

if k describe pod -n kube-system kube-proxy-csjj6:
Tolerations:                 op=Exists
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
but why it's run on master? because kube-proxy is deamon set and daemon set run on all nodes
k describe pod -n kube-system kube-apiserver-k8s-master1
Tolerations:       :NoExecute op=Exists => every node has NoExecute this pod can tolerate like static pods

static pod => pods is master => ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

for e normal pod like coredns:
Tolerations:                 CriticalAddonsOnly op=Exists
                             node-role.kubernetes.io/control-plane:NoSchedule
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s


how to add taint on node:
k taint node k8s-worker2 node-type=production:NoSchedule

at first, just master has taint


NodeAffinity => pod run on which node
AntiNodeAffinity => pod should not run on which node
PodAffinity => pod run on node that has another pod with specific spec
PodAntiAffinity => pod should not run on node that has another pod with specific spec

-------------------------------------------

CRD or Custom Resource Definition
Extend api server for doing something in whole cluster
for example we can use kubectl get monitoring and so on
install elk without custom resource is complicated
we can use operator that create crd for interact with api server

what is operator ? for example operator is stateful pod
first we create a crd and then create operator for watch this new resource
crd create a new apiVersion and kind then we use them


-----------------------------------------------
HPA Horizontal Pod Auto scale:
k8s auto scale replica(we learned fixed replica)
k8s detemined this with some thresold and metrics like cpu

desiredReplica = ceil[currentReplica * (currentMetricValue / desiredMetricValue)]

for that, we use HorizentalPodAutoscaler
also we need metric server for k8s to determind metrics like kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.0/components.yaml

then we can use:
k top pod -A

note:
for metric server we shoud add this two line before apply:
in spec.template.spec.containers[0].args:
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP

or after apply use edit:
kubectl -n kube-system edit deploy metrics-server

---------------------------------
cni container network interface
calico,flannel and so on

network in kubernetes is so important
calico per pod create a virtual interface outside of pod(naming calico1234...) and create a virtual router in each node
this router and outer interface locate between internal interface of pod and real interface of node
in ClusterIP service:
calico in every request doing nat and change destination and so on but source ip not change
note: nat do in calico outer interface

in NodePort service:
a client outside of cluster call a service and everything is same

------------------------------------------
helm:
one of most important task of helm is package manager

Helm chart Include this main file:
Chart.yaml just description and definiton about chart
template dir: containt services, configs and other resource
values.yaml : configs of resources

first we need install helm:
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version

or

curl -LO https://get.helm.sh/helm-v4.0.2-linux-amd64.tar.gzk        (if first test previous method we can get this url)
tar -zxvf helm-v4.0.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
sudo chmod +x /usr/local/bin/helm
helm version

then create helm-repo-proxy in nexus with this url:
https://charts.bitnami.com/bitnami

then add repo:
helm repo add helm-repo-proxy http://192.168.211.1:8081/repository/helm-repo-proxy/
helm install mypostgres bitnami/postgresql -n hnp

another way download chart(folder of chart-for example mydb_dir)
then 
helm install mypostgres ./mydb_dir
then:
helm list
helm uninstall my-tomcat

------------------------------------------