install nginx ingress controller with manifest

first download repo:
git clone https://github.com/nginx/kubernetes-ingress.git --branch v5.2.1

cd kubernetes-ingress/

kubectl apply -f deployments/common/ns-and-sa.yaml
kubectl apply -f deployments/rbac/rbac.yaml
kubectl apply -f examples/shared-examples/default-server-secret/default-server-secret.yaml
kubectl apply -f deployments/common/nginx-config.yaml
kubectl apply -f deployments/common/ingress-class.yaml

check created class:
k get ingressclasses.networking.k8s.io


kubectl apply -f config/crd/bases/k8s.nginx.org_virtualservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_transportservers.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_policies.yaml
kubectl apply -f config/crd/bases/k8s.nginx.org_globalconfigurations.yaml


deploying NGINX Ingress Controller:
kubectl apply -f deployments/deployment/nginx-ingress.yaml


k get deployments.apps -A --------> nginx-ingress
k get pod -n nginx-ingress



access nginx controller:
kubectl create -f deployments/service/nodeport.yaml
check:
k get service -n nginx-ingress




how to delete all installed:
kubectl delete namespace nginx-ingress
k delete ingressclasses.networking.k8s.io nginx
....



install version 1.x:
