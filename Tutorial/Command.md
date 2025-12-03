vim ~/.bashrc
alias k='kubectl'
source <(kubectl completion bash)
complete -F __start_kubectl k

then source ~/.bashrc

kubectl get <resource-name> -A or --namespace <name> or -n 


kubectl get po -A
kubectl get po -A -o wide

kubectl get node -A

-- check services:
kubectl get services or svc

-- create pod
kubectl run nginx-pod --image nginx:latest --restart=Never --port=80 -n default
kubectl describe pod nginx-pod -n default

kubectl get pods -n default

kubectl logs nginx-pod -n default
kubectl logs --previous nginx-pod -n default

kubectl expose pod nginx-pod --type=NodePort --port=80 --name=nginx-service

kubectl delete pod nginx-pod -n default
kubectl delete service nginx-service -n default



--- pod with manifest:
kubectl create -f nginx.yaml
k delete -f nginx.yaml



k get po <pod name> -n <namespace> -o yaml (show manifest) or json

-- namespace
k get namespace (ns)
k create namespace test


-------------
logs
k logs -n default nginx-pod
k logs -n default nginx-pod -c nginx-container (-f)

---------------
edit pod
k edit pod -n default nginx-pod


-----------------
access to container without expose port(just for local access)
k port-forward -n default nginx-pod 8888:80


-- show labels
k get pod -n hnp --show-labels

-- add label
k label po -n hnp nginx-pod app=ui (--overwrite)


-- show by label
k get po -n hnp -L app,rel

k get pod -n hnp -l app=item,rel=stable

k get pod -n hnp -l '!rel',app=payment


-- can set label on all resource
k label node k8s-worker1 disk=ssd

k get pod nginx-pod -n hnp -o yaml
-- can set annotate:
k annotate po nginx-pod -n hnp author="Hadi HNP"


-- delete pod with label and namespace
k delete pod -n hnp -l app=item
k delete pod -n hnp --all

-- delete namespace
k delete ns -n hnp

-- delete all resource in namespace
k delete all --all -n hnp


-- watch namespace:
k get pod -w -n hnp

-- replication controller (for replica set using rs)
k get rc
k delete rc nginx-rc or k delete -f nginx-rc.yaml

-- check node
systemctl status containerd
systemctl status kubelet


-- daemonset
k get ds

-- job
k get job

-- cron job
k get cronjobs.batch



-- run dnsutils for test HeadLessService
kubectl run dnsutils -n hnp --image=tutum/dnsutils --command -- sleep infinity
kubectl exec -n hnp dnsutils -- nslookup nginx-headless


-- access to a contianer of pod(if not specify -c, then connect to first container)
kubectl exec -it -n hnp my-pod-vol -c my-pod-vol-1 -- sh


-- how to check kubelet logs on nodes:
sudo journalctl -u kubelet -f


-- access longhorn dashboard outside of cluster just for test:
on master:
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
on local system:
ssh -L 8080:localhost:8080 user@192.168.211.131
http://localhost:8080


-- how to check node port service connect to endpoint:
kubectl get endpoints -n longhorn-system


-- create config map with command
k create configmap nginx-config --from-literal=port=8888 --from-literal=abc=xyz

or create nginx-conf.conf:
port=8888
abc=xyz

then 
k create configmap nginx-config --from-file=nginx-conf.conf
k get cm nginx-conf -o yaml

we can edit config map at runtime and after few seccond, it effects on container(if we not use subPath):
k edit configmaps -n hnp nginx-config
edit with vim and save


-- use secrets and set ssl for nginx
create key:
openssl genrsa -out https.key 2048
create certificate
openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.hnptest.com

echo one > two

k create secret generic nginx-https --from-file=https.key --from-file=https.cert --from-file=two -n hnp

also we have image pull secret, we can set this secret for deoployment or replica and so on, to use this user pass for pull image from repository



-- get data about cluster ip and coredns:
kubectl cluster-info
kubectl cluster-info dump => get more details

after find ip we can use curl to connect api server:
curl https://192.168.211.131:6443 -k
but it says 403!
for now we can use kubectl proxy and connect to api server with proxy:
 kubectl proxy
 then we can use all api
 for example get list of jobs:
 curl localhost:8001/apis/batch/v1/jobs
 or list of replicaset:
 curl localhost:8001/apis/apps/v1/replicasets


also we can use curl image(Manifest/Metadata/CurlPodApiServer.yaml)
we run k get svc
and find kubernetes service name
then exec to curl container(k exec -it -n hnp test -- sh)
run
env
and find KUBERNETES_SERVICE_HOST (is k8s ip for connection to api server)
curl kubernetes.default -v -> we get certificate problem
in path ls /var/run/secrets/kubernetes.io/serviceaccount/ we have ca.crt and use this certificate for connection
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default
again we get 403
we have a token in this path: /var/run/secrets/kubernetes.io/serviceaccount/token
create env for simplicity:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
and finally  we need enable cluster role binding for anonymos user:
kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous

then:
curl -H "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default

but it has security problem and we should delete clusterrolebinding after doing own work:
k delete clusterrolebindings.rbac.authorization.k8s.io cluster-system-anonymous



---------------------------------
indeed deployment is replicaset + versioning
after create a deployment we can change version of image and apply again
we can use below command to see history:
k rollout history deployment -n hnp
k rollout status deployment -n hnp

if we use --record true in creation deployment we can see cause changes in status:
k apply -f nginxdp.yaml --record=true

we can undo:
k rollout undo -n hnp deployment nginx-dp

we can restart using k rollout restart and so on

back to specefic version(find revision using history):
k rollout undo -n hnp deployment nginx-dp --to-revision=1

--------------------------------

every namespace has a default service account that manage communication between pods
service accound is a resource
k get serviceaccounts -n hnp
k get serviceaccounts -n hnp -o yaml

we can use jq command:
k get pod -n hnp -o json | jq .items

how to see token of service account in container:
k exec -it -n hnp test -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

