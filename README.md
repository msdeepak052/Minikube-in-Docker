# Minikube-in-Docker

# Installing Minikube in Docker and Running an Nginx Pod

Here's a step-by-step guide to install Minikube inside Docker and then run an Nginx pod that's accessible from your host machine.

## Prerequisites
- Docker installed on your host machine
- Sufficient resources (CPU, memory) allocated to Docker

## Step 1: Install Minikube inside Docker

1. **Pull the Minikube Docker image**:
   ```bash
   docker pull minikube/minikube
   ```

2. **Start Minikube container with Docker-in-Docker (DinD)**:
   ```bash
   docker run -d --privileged --name minikube \
     -v /var/lib/minikube:/var/lib/minikube \
     -v /var/run/docker.sock:/var/run/docker.sock \
     minikube/minikube start --driver=docker --force-systemd=true
   ```

3. **Enter the Minikube container**:
   ```bash
   docker exec -it minikube bash
   ```

## Step 2: Verify Minikube Installation

Inside the container:
```bash
minikube status
kubectl version
```

## Step 3: Create an Nginx Pod

1. **Create a deployment**:
   ```bash
   kubectl create deployment nginx --image=nginx
   ```

2. **Expose the deployment as a service**:
   ```bash
   kubectl expose deployment nginx --type=NodePort --port=80
   ```

3. **Find the service details**:
   ```bash
   kubectl get svc nginx
   ```
   Note the NodePort (something like 3xxxx) assigned to the service.

## Step 4: Access Nginx from Host Machine

1. **Find the Minikube container's IP address**:
   On your host machine (not inside the container), run:
   ```bash
   docker inspect minikube | grep IPAddress
   ```
   Note the IP address (typically something like 172.17.0.2).

2. **Access Nginx**:
   Use the IP address and NodePort in your browser or curl:
   ```bash
   curl http://<container-ip>:<node-port>
   ```
   Example:
   ```bash
   curl http://172.17.0.2:32456
   ```

## Alternative: Using Minikube Tunnel (for LoadBalancer)

If you prefer to use LoadBalancer type:

1. Inside the container, create a LoadBalancer service:
   ```bash
   kubectl expose deployment nginx --type=LoadBalancer --port=80
   ```

2. In a separate terminal, run the tunnel:
   ```bash
   docker exec -it minikube minikube tunnel
   ```

3. Get the external IP:
   ```bash
   kubectl get svc nginx
   ```
   Then access it with the external IP on port 80.

## Clean Up

When you're done:
```bash
# Delete the service and deployment
kubectl delete svc nginx
kubectl delete deployment nginx

# Stop and remove the Minikube container
docker stop minikube
docker rm minikube
```

This setup allows you to run Kubernetes workloads in an isolated Docker container while still being able to access them from your host machine.
