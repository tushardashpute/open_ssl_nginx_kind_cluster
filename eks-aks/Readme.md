Step1 . create  nginx ingress

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm install my-ingress-nginx ingress-nginx/ingress-nginx --version 4.9.1

Step2 : Deploy sample app

kubectl apply -f deploy1.yaml
kubectl apply -f deploy2.yaml

Step3 : Create nginx 

kubectl apply -f ingress_without_ssl.yaml

Step4 : create ssl
