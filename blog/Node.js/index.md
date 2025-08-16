---
title: Debian Node.js Deployment
date: 2025-8-16
description: This guide will help deploy Node.js on Debian with Kubernetes
categories: [Tutorials]
draft: false # Change to true to not render the post in on the website
---

<h3>**What is Node.js**</h3>

Node.js is a JavaScript runtime for running JS outside the browser—most often on servers. It’s built on Chrome’s V8 engine and uses an event-driven, non-blocking I/O model (via libuv), which makes it fast and efficient for network and I/O-heavy work (APIs, real-time apps, proxies).

<h3>**Why Should You Use Node.js**</h3>

1. Concurrency without the headache: Event-driven, non-blocking I/O lets one process handle thousands of connections (perfect for REST/GraphQL, WebSockets, streaming).
2. Single language everywhere: Frontend + backend in JS/TS means shared types and models, faster teams, less context switching.
3. Massive ecosystem: npm gives you libraries for almost everything (auth, ORM, queues, observability). You’ll ship faster.
4. Fast iteration: Great DX—hot reloaders, batteries-included tooling, lots of tutorials.
5. Cloud/K8s friendly: Small base image (e.g., node:18-alpine), quick cold starts, plays well with containers, horizontal scaling is easy.
6. Real-time: Socket.io, SSE, and event streams are very natural in Node.


<h3>**The Steps**</h3>

In order to deploy Node.js on Debian with Kubernetes you are going to need to do the following:

**1) Install and configure containerd**
<pre># Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Generate default config, then tweak for Kubernetes
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Use systemd cgroups and the recommended pause image
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#\[plugins."io.containerd.grpc.v1.cri"\]#\[plugins."io.containerd.grpc.v1.cri"\]\n  sandbox_image = "registry.k8s.io/pause:3.10"#' /etc/containerd/config.toml

# Start containerd
sudo systemctl enable --now containerd
</pre>

**2) Kernel settings and swap**
<pre>
# Required sysctls
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo modprobe br_netfilter
sudo sysctl --system

# Disable swap (Kubernetes requirement)
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
</pre>

**3) Install Kubernetes binaries (kubeadm, kubelet, kubectl)**
<pre>
# Modern repo for Debian 12 (bookworm)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl cri-tools
sudo apt-mark hold kubelet kubeadm kubectl
</pre>

**4) (Optional) Pre-pull control-plane images**
<pre>
sudo kubeadm config images pull --kubernetes-version v1.33.4
</pre>

**5) Initialize a single-node control plane**
<pre>
# Choose a pod CIDR compatible with your CNI (here we’ll use Flannel)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
</pre>

When it finishes, configure kubectl for your user:
<pre>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
</pre>

**6) Install a CNI (Flannel example)**
<pre>
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml
</pre>

Verify the node is Ready and CoreDNS is Running:
<pre>
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
</pre>

**7) Deploy a minimal Node.js service**
Create a single YAML file:
<pre>
vim > nodejs-hello.yaml <<'YAML'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodejs-hello
data:
  server.js: |
    const http = require('http');
    const port = process.env.PORT || 3000;
    const server = http.createServer((req, res) => {
      res.writeHead(200, {'Content-Type':'application/json'});
      res.end(JSON.stringify({ ok: true, path: req.url, ts: new Date().toISOString() }));
    });
    server.listen(port, () => console.log(`listening on ${port}`));
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-hello
  template:
    metadata:
      labels:
        app: nodejs-hello
    spec:
      containers:
      - name: node
        image: node:18-alpine
        command: ["node","/app/server.js"]
        ports:
        - containerPort: 3000
        resources:
          requests: { cpu: 50m, memory: 64Mi }
          limits:   { cpu: 250m, memory: 128Mi }
        readinessProbe:
          httpGet: { path: "/", port: 3000 }
          initialDelaySeconds: 3
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: "/", port: 3000 }
          initialDelaySeconds: 10
          periodSeconds: 10
        volumeMounts:
        - name: app-src
          mountPath: /app
        env:
        - name: PORT
          value: "3000"
      volumes:
      - name: app-src
        configMap:
          name: nodejs-hello
          items: [{ key: server.js, path: server.js }]
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-hello
  
spec:
  type: NodePort
  selector:
    app: nodejs-hello
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30008
YAML
</pre>

Apply and Check:
<pre>
kubectl apply -f nodejs-hello.yaml
kubectl get pods -o wide
kubectl get svc nodejs-hello -o wide
</pre>

Test from another machine on the network (or from the node itself):
<pre>
http://<NODE-IP>:30008/
</pre>

You should see:
<pre>
{ "ok": true, "path": "/", "ts": "..." }
</pre>

**8) Useful day-2 commands**
<pre>
# Scale
kubectl scale deploy/nodejs-hello --replicas=3

# Update image (use your own registry/repo:tag)
kubectl set image deploy/nodejs-hello node=<your-registry>/<repo>:<tag>
kubectl rollout status deploy/nodejs-hello

# Logs
kubectl logs -l app=nodejs-hello

# Clean up
kubectl delete -f nodejs-hello.yaml
</pre>

**9) Check if the service is deployed**
<pre>
kubectl get nodes -o wide
</pre>

That is the way to deploy Node.js on Debian with Kubernetes.



