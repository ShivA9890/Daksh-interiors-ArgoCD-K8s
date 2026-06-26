INSTALL argoCD

create namespace argoCD

kubectl apply -f namespace.yml

install argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

check all the pods are running

kubectl get pods -n argocd

unlock the dashboard

kubectl port-forward svc/argocd-server -n argocd 8083:443 &


get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

usernme: admin
use admin password
login
change password


