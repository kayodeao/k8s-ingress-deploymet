# Kubernetes Deployment with Ingress and TLS

This guide covers the steps to deploy an application to Kubernetes, set up Ingress for routing traffic, manage TLS certificates with cert-manager.

## Prerequisites
- A Kubernetes cluster
- `kubectl` configured
- `Helm` installed
- Domain name
- DNS provider access

## Steps

### Step 1: Deploy the Application

1. Create a Kubernetes deployment file (`deployment.yaml`):

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
      labels:
        app: my-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: my-app
      template:
        metadata:
          labels:
            app: my-app
        spec:
          containers:
          - name: my-app
            image: my-app-image:latest
            ports:
            - containerPort: 80
    ```

2. Apply the deployment file:

    ```sh
    kubectl apply -f deployment.yaml
    ```

3. Create a service file (`service.yaml`):

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-app-service
    spec:
      selector:
        app: my-app
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

4. Apply the service file:

    ```sh
    kubectl apply -f service.yaml
    ```

### Step 2: Set Up Ingress for Routing Traffic

1. Install the NGINX Ingress Controller using Helm:
     ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm search repo ingress-nginx --versions
    ```

From the app version we select the version that matches the compatibility matrix. </br>

    ```
    NAME                            CHART VERSION   APP VERSION     DESCRIPTION
    ingress-nginx/ingress-nginx     4.4.0           1.5.1           Ingress controller for Kubernetes using NGINX a...
    ```

Now we can use `helm` to install the chart directly if we want. </br>
Or we can use `helm` to grab the manifest and explore its content. </br>
We can also add that manifest to our git repo if we are using a GitOps workflow to deploy it. </br>

    ```
    CHART_VERSION="4.4.0"
    APP_VERSION="1.5.1"
    
    mkdir ./kubernetes/ingress/controller/nginx/manifests/
    
    helm template ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --version ${CHART_VERSION} \
    --namespace ingress-nginx \
    > ./kubernetes/ingress/controller/nginx/manifests/nginx-ingress.${APP_VERSION}.yaml
    ```

2. Create an Ingress resource file (`ingress.yaml`):

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-app-ingress
      annotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod"
    spec:
      tls:
      - hosts:
        - my-app.example.com
        secretName: my-app-tls
      rules:
      - host: my-app.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
    ```

3. Apply the Ingress resource:

    ```sh
    kubectl apply -f ingress.yaml
    ```

### Step 3: Set Up cert-manager for TLS Certificates

1. Install cert-manager:

    ```sh
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
    ```

2. Create a ClusterIssuer resource for Let's Encrypt (`cluster-issuer.yaml`):

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: nginx
    ```

3. Apply the ClusterIssuer resource:

    ```sh
    kubectl apply -f cluster-issuer.yaml
    ```

### Step 4: Create a DNS Record

1. Identify the IP address of the NGINX Ingress Controller's Load Balancer:

    ```sh
    kubectl get svc -n ingress-nginx
    ```

    CHECK for the EXTERNAL-IP of the `ingress-nginx-controller` service.

2. Create a record in your DNS provider's management console pointing to this IP. 

### Step 5: Verify Deployment

1. Ensure all resources are deployed:

    ```sh
    kubectl get all
    ```

2. Access your application via the browser at `https://my-app.example.com`.

3. Check the certificate status:

    ```sh
    kubectl describe certificate my-app-tls
    ```






