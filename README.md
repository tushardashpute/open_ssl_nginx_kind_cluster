# open_ssl_nginx_kind_cluster

**Step 1: Create Kind Cluster**

Install and start Docker daemon

        yum install docker -y && service docker start

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
        EOF



