# open_ssl_nginx_kind_cluster

**Step 1: Create Kind Cluster**

Install and start Docker daemon

        yum install docker -y && service docker start

Install kubectl 

        curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.17/2023-11-14/bin/linux/amd64/kubectl
        chmod +x kubectl
        mv kubectl /usr/local/bin/kubectl

Install Kind on Linux

        curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64
        chmod +x ./kind
        sudo mv ./kind /bin/kind
        
Create a kind cluster with extraPortMappings and node-labels.

extraPortMappings allow the local host to make requests to the Ingress controller over ports 80/443
node-labels only allow the ingress controller to run on a specific node(s) matching the label selector

        cat <<EOF | kind create cluster --config=-
        kind: Cluster
        apiVersion: kind.x-k8s.io/v1alpha4
        nodes:
        - role: control-plane
          kubeadmConfigPatches:
          - |
            kind: InitConfiguration
            nodeRegistration:
              kubeletExtraArgs:
                node-labels: "ingress-ready=true"
          extraPortMappings:
          - containerPort: 80
            hostPort: 80
            protocol: TCP
          - containerPort: 443
            hostPort: 443
            protocol: TCP
        - role: worker
        - role: worker
        EOF

The output shows the progress of the operation. When the cluster successfully initiates, the command prompt appears.

![image](https://github.com/tushardashpute/open_ssl_nginx_kind_cluster/assets/74225291/b00fb1fe-10c5-4d27-bafe-f210edf67d34)

        kubectl get nodes
        NAME                 STATUS   ROLES           AGE   VERSION
        kind-control-plane   Ready    control-plane   47s   v1.24.0
        kind-worker          Ready    <none>          26s   v1.24.0
        kind-worker2         Ready    <none>          26s   v1.24.0


**Step 2: Deploy nginx Ingress controller**

        $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
        
        $ kubectl wait --namespace ingress-nginx \
          --for=condition=ready pod \
          --selector=app.kubernetes.io/component=controller \
          --timeout=90s
        
        ...
        
        pod/ingress-nginx-controller-6f9b5dd966-zkj2c condition met

Check if it works

# apply ingress test manifests

        kubectl apply -f https://raw.githubusercontent.com/tushardashpute/open_ssl_nginx_kind_cluster/main/sample_app.yaml

# Check the pod,service,ingress

        kubectl get ing,svc,pod
        NAME                                        CLASS    HOSTS           ADDRESS     PORTS   AGE
        ingress.networking.k8s.io/example-ingress   <none>   ingress.local   localhost   80      45s
        
        NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
        service/bar-service   ClusterIP   10.96.18.173    <none>        8080/TCP   45s
        service/foo-service   ClusterIP   10.96.246.180   <none>        8080/TCP   46s
        
        NAME          READY   STATUS    RESTARTS   AGE
        pod/bar-app   1/1     Running   0          46s
        pod/foo-app   1/1     Running   0          46s
To make ingress.local available in local browser, add to the /etc/hosts line:

127.0.0.1 ingress.local – in my case cluster is running at 127.0.0.1.

# test ingress
curl ingress.local/foo/hostname
foo-app
curl ingress.local/bar/hostname
bar-app









