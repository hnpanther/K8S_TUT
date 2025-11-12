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
