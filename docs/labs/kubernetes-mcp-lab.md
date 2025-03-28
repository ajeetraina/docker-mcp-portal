---
layout: lab
title: Lab 5: Deploying MCP Servers on Kubernetes
description: Learn how to deploy Docker MCP servers in a Kubernetes environment to enable scalable AI capabilities in your cluster.
difficulty: Advanced
time: 75 minutes
author: Docker Team
last_updated: March 18, 2025
prev_lab: /docs/labs/production-deployment-lab
next_lab: /docs/labs/ai-powered-cicd-lab
---

<div class="lab-prerequisites">
  <h2><i class="fas fa-clipboard-list"></i> Prerequisites</h2>
  <ul>
    <li>Kubernetes cluster (local like minikube/kind or cloud-based)</li>
    <li>kubectl CLI tool installed and configured</li>
    <li>Helm installed (v3.x+)</li>
    <li>Basic understanding of Kubernetes concepts</li>
    <li>Completion of previous MCP labs recommended</li>
  </ul>
</div>

<div class="learning-objectives">
  <h2><i class="fas fa-graduation-cap"></i> Learning Objectives</h2>
  <ol>
    <li>Deploy MCP servers as Kubernetes deployments</li>
    <li>Configure Kubernetes services and ingress for MCP servers</li>
    <li>Implement persistent storage for MCP servers that require it</li>
    <li>Set up proper security constraints using Kubernetes primitives</li>
    <li>Manage MCP server configuration with ConfigMaps and Secrets</li>
  </ol>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-cube"></i> Step 1: Create a Kubernetes Namespace for MCP Servers
  </div>
  <div class="lab-step-content">
    <p>Let's start by creating a dedicated namespace for our MCP servers:</p>

```bash
kubectl create namespace mcp-system
kubectl config set-context --current --namespace=mcp-system
```

    <p>This keeps our MCP resources organized and separate from other applications in the cluster.</p>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-clock"></i> Step 2: Deploy the Time MCP Server
  </div>
  <div class="lab-step-content">
    <p>Let's start with a simple MCP server deployment. Create a file named <code>time-mcp-deployment.yaml</code>:</p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: time-mcp
  labels:
    app: time-mcp
    component: mcp-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: time-mcp
  template:
    metadata:
      labels:
        app: time-mcp
    spec:
      containers:
      - name: time-mcp
        image: mcp/time:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "0.2"
            memory: "256Mi"
          requests:
            cpu: "0.1"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /.well-known/mcp
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /.well-known/mcp
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: time-mcp
  labels:
    app: time-mcp
    component: mcp-server
spec:
  selector:
    app: time-mcp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

    <p>Apply the configuration:</p>

```bash
kubectl apply -f time-mcp-deployment.yaml
```

    <p>Verify the deployment:</p>

```bash
kubectl get pods -l app=time-mcp
kubectl get service time-mcp
```

    <div class="lab-note">
      <h4><i class="fas fa-info-circle"></i> Understanding the Configuration</h4>
      <p>We've configured the Time MCP server with:</p>
      <ul>
        <li><strong>Multiple replicas</strong> for high availability</li>
        <li><strong>Resource limits</strong> to prevent resource contention</li>
        <li><strong>Health checks</strong> so Kubernetes can monitor the service health</li>
        <li>A <strong>ClusterIP Service</strong> to provide internal access to the MCP server</li>
      </ul>
    </div>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-database"></i> Step 3: Deploy the Filesystem MCP Server with Persistent Storage
  </div>
  <div class="lab-step-content">
    <p>The Filesystem MCP server requires persistent storage. Let's set that up:</p>

    <p>First, create a PersistentVolumeClaim in <code>filesystem-mcp-pvc.yaml</code>:</p>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: filesystem-mcp-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

    <p>Apply the PVC:</p>

```bash
kubectl apply -f filesystem-mcp-pvc.yaml
```

    <p>Now create the deployment in <code>filesystem-mcp-deployment.yaml</code>:</p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: filesystem-mcp
  labels:
    app: filesystem-mcp
    component: mcp-server
spec:
  replicas: 1  # Only one replica since we're using a ReadWriteOnce volume
  selector:
    matchLabels:
      app: filesystem-mcp
  template:
    metadata:
      labels:
        app: filesystem-mcp
    spec:
      containers:
      - name: filesystem-mcp
        image: mcp/filesystem:latest
        args:
        - /data
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /data
        resources:
          limits:
            cpu: "0.3"
            memory: "384Mi"
          requests:
            cpu: "0.1"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /.well-known/mcp
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: filesystem-mcp-data
---
apiVersion: v1
kind: Service
metadata:
  name: filesystem-mcp
  labels:
    app: filesystem-mcp
    component: mcp-server
spec:
  selector:
    app: filesystem-mcp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

    <p>Apply the deployment:</p>

```bash
kubectl apply -f filesystem-mcp-deployment.yaml
```

    <p>Let's add some data to the filesystem for testing:</p>

```bash
# Find the pod name
FS_POD=$(kubectl get pods -l app=filesystem-mcp -o jsonpath='{.items[0].metadata.name}')

# Create a test file
kubectl exec $FS_POD -- sh -c 'echo "This is a test file for our Kubernetes MCP lab" > /data/test.txt'

# Verify the file was created
kubectl exec $FS_POD -- cat /data/test.txt
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-globe"></i> Step 4: Deploy the Fetch MCP Server
  </div>
  <div class="lab-step-content">
    <p>The Fetch MCP server requires internet access. Create <code>fetch-mcp-deployment.yaml</code>:</p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fetch-mcp
  labels:
    app: fetch-mcp
    component: mcp-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fetch-mcp
  template:
    metadata:
      labels:
        app: fetch-mcp
    spec:
      containers:
      - name: fetch-mcp
        image: mcp/fetch:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: "0.3"
            memory: "384Mi"
          requests:
            cpu: "0.1"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /.well-known/mcp
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: fetch-mcp
  labels:
    app: fetch-mcp
    component: mcp-server
spec:
  selector:
    app: fetch-mcp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

    <p>Apply the configuration:</p>

```bash
kubectl apply -f fetch-mcp-deployment.yaml
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-shield-alt"></i> Step 5: Create an API Gateway for Secure MCP Access
  </div>
  <div class="lab-step-content">
    <p>Let's create an API Gateway to securely expose our MCP servers. Create <code>mcp-ingress.yaml</code>:</p>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcp-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: mcp-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  rules:
  - http:
      paths:
      - path: /time(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: time-mcp
            port:
              number: 80
      - path: /fs(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: filesystem-mcp
            port:
              number: 80
      - path: /fetch(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: fetch-mcp
            port:
              number: 80
```

    <p>Create a basic authentication secret:</p>

```bash
# Create credentials file with username:password format (mcp:mcppassword)
# Using 'mcp' as username and 'mcppassword' as password
htpasswd -c auth mcp
# Enter 'mcppassword' when prompted

# Create the Kubernetes secret
kubectl create secret generic mcp-basic-auth --from-file=auth

# Clean up the temporary file
rm auth
```

    <p>Apply the ingress configuration:</p>

```bash
kubectl apply -f mcp-ingress.yaml
```

    <p>Get the ingress address:</p>

```bash
kubectl get ingress mcp-gateway
```

    <div class="lab-note">
      <h4><i class="fas fa-info-circle"></i> Note</h4>
      <p>Depending on your Kubernetes environment, you may need to install an Ingress Controller like NGINX Ingress Controller before the Ingress resource will work properly. For local development with minikube, you can enable the NGINX ingress addon with <code>minikube addons enable ingress</code>.</p>
    </div>
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-key"></i> Step 6: Set Up Secure Secrets Management for MCP Servers
  </div>
  <div class="lab-step-content">
    <p>For MCP servers that require API keys, like GitHub or PostgreSQL, we should use Kubernetes Secrets:</p>

```bash
# Create a Secret for GitHub token
kubectl create secret generic github-mcp-secret \
  --from-literal=GITHUB_TOKEN=your_github_token

# Create a Secret for PostgreSQL credentials
kubectl create secret generic postgres-mcp-secret \
  --from-literal=POSTGRES_USER=mcpuser \
  --from-literal=POSTGRES_PASSWORD=securepassword \
  --from-literal=POSTGRES_DB=mcpdb
```

    <p>Now deploy the GitHub MCP server with the secret. Create <code>github-mcp-deployment.yaml</code>:</p>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-mcp
  labels:
    app: github-mcp
    component: mcp-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-mcp
  template:
    metadata:
      labels:
        app: github-mcp
    spec:
      containers:
      - name: github-mcp
        image: mcp/github:latest
        ports:
        - containerPort: 8080
        envFrom:
        - secretRef:
            name: github-mcp-secret
        resources:
          limits:
            cpu: "0.3"
            memory: "384Mi"
          requests:
            cpu: "0.1"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: github-mcp
  labels:
    app: github-mcp
    component: mcp-server
spec:
  selector:
    app: github-mcp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

    <p>Apply the GitHub MCP server:</p>

```bash
kubectl apply -f github-mcp-deployment.yaml
```

    <p>Update the ingress to include the GitHub MCP server:</p>

```bash
# Add this path to your mcp-ingress.yaml file under the paths section
# and then reapply the configuration
      - path: /github(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: github-mcp
            port:
              number: 80
```

```bash
kubectl apply -f mcp-ingress.yaml
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-robot"></i> Step 7: Configure Gordon AI to Use Kubernetes MCP Servers
  </div>
  <div class="lab-step-content">
    <p>Now let's create a <code>gordon-mcp.yml</code> file that uses our Kubernetes-hosted MCP servers:</p>

```yaml
services:
  k8s-time-mcp:
    image: curlimages/curl
    command: |
      -s -X GET -f 
      -u mcp:mcppassword 
      http://YOUR_INGRESS_IP/time/.well-known/mcp
    labels:
      mcp.base-url: http://YOUR_INGRESS_IP/time/
      mcp.auth: Basic bWNwOm1jcHBhc3N3b3Jk  # Base64 encoded mcp:mcppassword

  k8s-fs-mcp:
    image: curlimages/curl
    command: |
      -s -X GET -f 
      -u mcp:mcppassword 
      http://YOUR_INGRESS_IP/fs/.well-known/mcp
    labels:
      mcp.base-url: http://YOUR_INGRESS_IP/fs/
      mcp.auth: Basic bWNwOm1jcHBhc3N3b3Jk

  k8s-fetch-mcp:
    image: curlimages/curl
    command: |
      -s -X GET -f 
      -u mcp:mcppassword 
      http://YOUR_INGRESS_IP/fetch/.well-known/mcp
    labels:
      mcp.base-url: http://YOUR_INGRESS_IP/fetch/
      mcp.auth: Basic bWNwOm1jcHBhc3N3b3Jk

  k8s-github-mcp:
    image: curlimages/curl
    command: |
      -s -X GET -f 
      -u mcp:mcppassword 
      http://YOUR_INGRESS_IP/github/.well-known/mcp
    labels:
      mcp.base-url: http://YOUR_INGRESS_IP/github/
      mcp.auth: Basic bWNwOm1jcHBhc3N3b3Jk
```

    <p>Replace <code>YOUR_INGRESS_IP</code> with the actual IP or hostname of your ingress.</p>
    
    <p>Test Gordon AI with your Kubernetes MCP servers:</p>

```bash
docker ai "What time is it now in New York, Tokyo, and London?"
```
  </div>
</div>

<div class="lab-step">
  <div class="lab-step-header">
    <i class="fas fa-cogs"></i> Step 8: Monitoring and Managing MCP Servers
  </div>
  <div class="lab-step-content">
    <p>Set up Prometheus monitoring for MCP servers by creating a ServiceMonitor (this requires Prometheus Operator):</p>

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mcp-servers
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      component: mcp-server
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

    <p>You can also create a dashboard in Grafana to visualize the metrics.</p>
    
    <p>To view the logs of your MCP servers:</p>

```bash
# For time MCP server
kubectl logs -l app=time-mcp

# For filesystem MCP server
kubectl logs -l app=filesystem-mcp
```
  </div>
</div>

<div class="lab-tip">
  <h4><i class="fas fa-lightbulb"></i> Best Practices for Kubernetes MCP Deployments</h4>
  <ul>
    <li><strong>Resource Management</strong>: Always set resource requests and limits</li>
    <li><strong>Health Checks</strong>: Implement proper liveness and readiness probes</li>
    <li><strong>Monitoring</strong>: Set up proper monitoring for all MCP servers</li>
    <li><strong>Security</strong>: Use secrets for sensitive information and network policies to restrict traffic</li>
    <li><strong>High Availability</strong>: Use multiple replicas where possible and consider Pod Disruption Budgets</li>
  </ul>
</div>

<div class="lab-note">
  <h4><i class="fas fa-exclamation-triangle"></i> Troubleshooting</h4>
  <p>Common issues and solutions:</p>
  <ul>
    <li>If pods are in a pending state, check for resource constraints or PVC binding issues</li>
    <li>If services aren't reachable, verify the service selectors match pod labels</li>
    <li>For ingress issues, check the ingress controller logs and ensure it's properly configured</li>
    <li>For authentication issues, verify the secrets are correctly created and mounted</li>
  </ul>
</div>

<div class="lab-conclusion">
  <h2><i class="fas fa-flag-checkered"></i> Conclusion</h2>
  <p>Congratulations! You've successfully deployed Docker MCP servers on Kubernetes, enabling your AI assistants to scale and run in your Kubernetes infrastructure. You've learned how to:</p>
  <ul>
    <li>Create Kubernetes deployments for MCP servers</li>
    <li>Configure services and ingress rules</li>
    <li>Set up persistent storage for stateful MCP servers</li>
    <li>Manage secrets securely</li>
    <li>Configure Gordon AI to use your Kubernetes-hosted MCP servers</li>
  </ul>
  <p>This approach allows you to leverage the scalability, resilience, and operational benefits of Kubernetes for your AI infrastructure.</p>
</div>

<div class="next-steps">
  <h2><i class="fas fa-arrow-circle-right"></i> Next Steps</h2>
  <p>Now that you've mastered deploying MCP servers on Kubernetes, you can:</p>
  <ul>
    <li><a href="/docs/labs/ai-powered-cicd-lab">Lab 6: Building AI-Powered CI/CD Pipelines with MCP</a> - Use Docker MCP to create intelligent CI/CD pipelines</li>
    <li>Explore implementing a service mesh like Istio for advanced networking capabilities</li>
    <li>Set up GitOps workflows for managing your MCP server deployments</li>
    <li>Integrate with cloud provider services for enhanced capabilities</li>
  </ul>
</div>