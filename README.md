# Kubernetes Deployment with Ingress and TLS

This guide covers the steps to deploy an application to Kubernetes, set up Ingress for routing traffic, manage TLS certificates with cert-manager, and create a DNS A record.

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

    ```sh
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx
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

### Step 4: Create a DNS A Record

1. Identify the IP address of the NGINX Ingress Controller's Load Balancer:

    ```sh
    kubectl get svc -n ingress-nginx
    ```

    Look for the EXTERNAL-IP of the `ingress-nginx-controller` service.

2. Create an A record in your DNS provider's management console pointing to this IP. 

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






