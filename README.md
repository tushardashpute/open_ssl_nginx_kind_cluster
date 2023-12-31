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

**Step 3 : Deploy and test sample application**

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

To make ingress.local available in local browser, add to the /etc/hosts line.

127.0.0.1 ingress.local – in my case cluster is running at 127.0.0.1.
public_ip ingress.local - if cluster is running on any cloud


![image](https://github.com/tushardashpute/open_ssl_nginx_kind_cluster/assets/74225291/7b3bf219-4599-40b0-bb8b-ea0abb9b07e1)


        # test ingress
        curl ingress.local/foo/hostname
        foo-app
        curl ingress.local/bar/hostname
        bar-app

if you want to access it from local browser, you can use public IP of instance.

**Step. 4 Add TLS encryption with self-signed certificate to enable HTTPs**

Until now, pod is exposed using Ingress, but the connection is over HTTP and therefore it is unencrypted. 
Let’s add some security to the server. First, create certifiates using openssl, then create kubernetes Secret of type ssl. 
And finally utilize it in Ingress resource.

1. Create self-signed certificate

        $ openssl req \
        -x509 -newkey rsa:4096 -sha256 -nodes \
        -keyout tls.key -out tls.crt \
        -subj "/CN=ingress.local" -days 365
        
        Generating a 4096 bit RSA private key
        ........++
        ...................................................................................................++
        writing new private key to 'tls.key'
        -----
        
        $ ls
        tls.crt  tls.key

2. Create kubernetes secret with those keys

        $ kubectl create secret tls ingress-local-tls \
          --cert=tls.crt \
          --key=tls.key
        
        secret/ingress-local-tls created

3. Make changes to Ingress

        cat <<EOF | k apply -f-
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: example-ingress
          annotations:
            nginx.ingress.kubernetes.io/rewrite-target: /$2
        spec:
          tls:                                # add those 4 lines
          - hosts:                            #
              - ingress.local                 #
            secretName: ingress-local-tls     #
          rules:
          - host: ingress.local
            http:
              paths:
              - pathType: Prefix
                path: /foo(/|$)(.*)
                backend:
                  service:
                    name: foo-service
                    port:
                      number: 8080
              - pathType: Prefix
                path: /bar(/|$)(.*)
                backend:
                  service:
                    name: bar-service
                    port:
                      number: 8080
        EOF





