# Minikube with 2 nodes - Kubernetes

Here is a complete, step-by-step guide for setting up **Minikube with 2 nodes**, **Helm**, and installing **Prometheus + Grafana**, including WSL configuration, port forwarding, and cleanup.

---

## 🧱 **Part 1: Setup Minikube with 2 Nodes (CPU: 4, RAM: 8196 MB)**

### 🔧 Step 1: Configure WSL2 to Use More Resources

Create or edit the `.wslconfig` file:

a. Open Notepad and paste:

   ```ini
   [wsl2]
   memory=10GB
   processors=4
   swap=4GB
   ```

b. Save it as:

   ```
   C:\Users\Deepak\.wslconfig
   ```

c. Restart WSL:

   ```bash
   wsl --shutdown
   ```
d. Confirm More Memory Is Available

In WSL terminal:

```bash
free -h
```

You should now see around **10 GB** total memory.

---
---

### ✅ 2. Install Minikube (Linux/WSL2)

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

---

### 🚀 Step 3: Start Minikube with 2 Nodes, 4 CPUs, 8192 MB RAM

Now that WSL can use more resources, run:

```bash
minikube delete  # optional clean start

minikube start --nodes 2 --cpus 4 --memory 8192 --driver=docker --cni=calico
```

If successful, verify:

```bash
minikube node list
kubectl get nodes
```
> Now you’ll have 1 control-plane + 1 worker.

Verify:

```bash
kubectl get nodes
```
---
> **⚡ Quick note:** Every time you restart your system, the Minikube cluster is not running automatically. You must run `minikube start` again to bring it up.

> You can make Minikube start automatically every time you launch Ubuntu in **WSL**.
> There are a couple of ways to do this depending on how you prefer to manage it:

---

## 🔹 Option 1: Add `minikube start` to `.bashrc` (runs every time you open a shell)

1. Open your `.bashrc`:

   ```bash
   nano ~/.bashrc
   ```

2. Add this at the bottom:

```bash
      # Auto-start Minikube when WSL Ubuntu launches
      if command -v minikube >/dev/null 2>&1; then
          # Check if a minikube profile already exists
          if ! minikube profile list 2>/dev/null | grep -q minikube; then
              echo "🚀 Creating a new Minikube cluster with Calico..."
              minikube start --nodes=2 --cpus=4 --memory=8192 --driver=docker --cni=calico
          elif ! minikube status >/dev/null 2>&1; then
              echo "▶️  Starting existing Minikube cluster..."
              minikube start
          else
              echo "✅ Minikube is already running."
          fi
      fi

```

   > This will only start Minikube if it’s **not already running**.

3. Save & exit (`CTRL+O`, `CTRL+X`).

4. Reload bash:

   ```bash
   source ~/.bashrc
   ```

---

## 🔹 Option 2: Use WSL startup commands (recommended, cleaner)

You can configure **Windows to run a command automatically when WSL Ubuntu starts**.

1. In **Windows**, open:

   ```
   C:\Users\<YourUser>\.wslconfig
   ```

   (Create it if it doesn’t exist.)

2. Add this section:

   ```ini
   [boot]
   command="wsl -d Ubuntu --exec bash -lc 'minikube status >/dev/null 2>&1 || minikube start --driver=docker'"
   ```

   * Replace `Ubuntu` with your WSL distro name (`wsl -l -v` to check).

3. Restart WSL:

   ```powershell
   wsl --shutdown
   wsl
   ```

Now, every time Ubuntu starts in WSL, it will auto-run `minikube start`.

---

## 🔹 Option 3: Systemd service (if your WSL has systemd enabled)

Since newer WSL supports **systemd**, you can create a service for Minikube.

1. Create a systemd service:

   ```bash
   sudo nano /etc/systemd/system/minikube.service
   ```
2. Add:

   ```ini
   [Unit]
   Description=Start Minikube on WSL boot
   After=network.target docker.service

   [Service]
   ExecStart=/usr/bin/minikube start --driver=docker
   ExecStop=/usr/bin/minikube stop
   Restart=always
   User=deepak

   [Install]
   WantedBy=default.target
   ```
3. Enable & start:

   ```bash
   sudo systemctl enable minikube --now
   ```

---

👉 Recommended for you: **Option 2 (wslconfig)** because it’s clean and doesn’t run `minikube start` every time you just open a new terminal tab.

---

## 🧱 **Part 2: Install Helm**

### ✅ 1. Install Helm

```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

### ✅ 2. Verify Helm

```bash
helm version
```

---

## 📦 **Part 3: Install Prometheus + Grafana (kube-prometheus-stack)**

### ✅ 1. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### ✅ 2. Create Namespace

```bash
kubectl create namespace monitoring
```
---

### ✅ 3. Install kube-prometheus-stack

```bash

helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./custom_kube_prometheus_stack.yml
```
#### custom_kube_prometheus_stack.yml

```
alertmanager:
  alertmanagerSpec:
    # Selects Alertmanager configuration based on these labels. Ensure that the Alertmanager configuration has matching labels.
    # ✅ Solves error: Misconfigured Alertmanager selectors can lead to missing alert configurations.
    # ✅ Solves error: Alertmanager wasn't able to findout the applied CRD (kind: Alertmanagerconfig)
    alertmanagerConfigSelector:
      matchLabels:
        release: monitoring

    # Sets the number of Alertmanager replicas to 3 for high availability.
    # ✅ Solves error: Single replica can cause alerting issues during pod failures.
    # ✅ Solves error: Alertmanager Cluster Status is Disabled (GitHub issue)
    replicas: 2

    # Sets the strategy for matching Alertmanager configurations. 'None' means no specific matching strategy.
    # ✅ Solves error: Incorrect matcher strategy can lead to unhandled alert configurations.
    # ✅ Solves error: Get rid of namespace matchers when creating AlertManagerConfig (GitHub issue)
    alertmanagerConfigMatcherStrategy:
      type: None
```
---

## 🌐 **Part 4: Access Prometheus & Grafana via Port-Forwarding**

### ✅ 1. Port Forward Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

* Open: [http://localhost:3000](http://localhost:3000)
* Get Grafana password:

  ```bash
  kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d
  ```

---

### ✅ 2. Port Forward Prometheus

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```

* Open: [http://localhost:9090](http://localhost:9090)

---

### ✅ 3. Port Forward Alertmanager

```bash
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring
```

* Open: [http://localhost:9093](http://localhost:9093)

---

To go **inside the Minikube nodes**, you can use the `minikube ssh` command.

---

## ✅ Accessing Minikube Nodes

### 🔹 1. **Go into the Control Plane Node**

```bash
minikube ssh
```

This defaults to the **primary control-plane node**.

---

### 🔹 2. **Go into a Specific Node (e.g., worker)**

First, list all nodes:

```bash
kubectl get nodes
```

You’ll see:

```
NAME           STATUS   ROLES           AGE     VERSION
minikube       Ready    control-plane   1h      v1.33.1
minikube-m02   Ready    <none>          10m     v1.33.1
```

Then SSH into a specific node like this:

```bash
minikube ssh -n minikube-m02
```

> Use the `-n` or `--node` flag to specify the node name.

---

## 🔙 Exit the Node

Once inside a node (you'll get a Linux shell), to exit:

```bash
exit
```

---

Let me know if you want to:

* Copy files into the node
* Run Docker inside the node (Minikube supports `docker` natively)


## 🧹 **Part 5: Cleanup Commands**

### ✅ Stop Minikube (pause state)

```bash
minikube stop
```

### ✅ Delete Minikube Cluster

```bash
minikube delete
```

This will remove all nodes, configurations, and containers.

---
