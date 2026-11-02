```markdown
# Production-Ready CI/CD Deployment on Multi-Node Kubernetes

An automated end-to-end DevOps pipeline for building, containerizing, and orchestrating a high-availability modern static web application using **Jenkins**, **Docker**, and **Kubernetes** on cloud-hosted Ubuntu infrastructure.

---

## 1. Architecture Overview

```text
[ Developer Local Machine ]
       │
       │  git push
       ▼
[ GitHub Repository ]
       │
       │  Webhook / SCM Poll
       ▼
[ Jenkins CI Server ] (Ubuntu 22.04 | 4 vCPU, 8 GB RAM)
       │
       ├─► Pull source code
       ├─► Build Docker image (Nginx:alpine based)
       ├─► Push image to Docker Hub
       └─► Execute kubectl deployment rollout via kubeconfig
             │
             ▼
[ Kubernetes Cluster ] (Private Subnet: 10.0.1.0/24)
       │
       ├── Control Plane: k8s-master (10.0.1.230 | Public: 154.21.235.229)
       │     └─► Manages cluster state, API server, etcd, scheduler
       │
       └── Data Plane: k8s-worker (10.0.1.176 | Public: 154.21.235.152)
             ├─► Calico CNI overlay network (192.168.0.0/16)
             ├─► 2x Replicated Pods (static-web-container)
             └─► NodePort Service exposed on port 32000
                   │
                   ▼
       [ End Users / Public Access: [http://154.21.235.152:32000](http://154.21.235.152:32000) ]

```

---

## 2. Infrastructure Topology

| Hostname | Role | OS | Flavor | Private IP | Public / Floating IP |
| --- | --- | --- | --- | --- | --- |
| **`jenkins-server`** | CI/CD Controller | Ubuntu 22.04 LTS | 4 vCPU, 8 GB RAM | `10.0.1.191` | `154.21.235.64` |
| **`k8s-master`** | Control Plane | Ubuntu 22.04 LTS | 4 vCPU, 8 GB RAM | `10.0.1.230` | `154.21.235.229` |
| **`k8s-worker`** | Worker Node | Ubuntu 22.04 LTS | 4 vCPU, 8 GB RAM | `10.0.1.176` | `154.21.235.152` |

---

## 3. Security Group Rules

### `sg-jenkins` (Jenkins Server)

* **Inbound:**
* `TCP 22` — SSH access (Administrator IP/32)
* `TCP 8080` — Jenkins Dashboard UI (`0.0.0.0/0` or Admin IP)


* **Outbound:**
* `ALL Traffic` — `0.0.0.0/0` (Pulls GitHub code, downloads packages, pushes to Docker Hub)



### `sg-k8s` (Master and Worker Nodes)

* **Inbound:**
* `TCP 22` — SSH access (Administrator IP/32)
* `ALL Traffic` — Subnet `10.0.1.0/24` (Intra-cluster communications: API server `6443`, etcd `2379-2380`, kubelet `10250`, Calico CNI `179/4789`)
* `TCP 6443` — `10.0.1.191` (Jenkins private IP for `kubectl` remote commands)
* `TCP 32000` (or `30000–32767`) — `0.0.0.0/0` (Public access to deployed NodePort application)


* **Outbound:**
* `ALL Traffic` — `0.0.0.0/0` (Pulls images from Docker Hub, updates system repositories)



---

## 4. Step-by-Step Implementation Guide

### A. Jenkins Server Configuration

```bash
# 1. Update packages and install Java 21 (Jenkins baseline requirement)
sudo apt update
sudo apt install -y openjdk-21-jdk curl wget apt-transport-https ca-certificates gnupg lsb-release

# 2. Add Jenkins repository and install Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc [https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key](https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key)
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] [https://pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable) binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins

# 3. Install Docker Engine
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin

# 4. Grant Docker permissions to Jenkins and Ubuntu users
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu

# 5. Install kubectl
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.30/deb/](https://pkgs.k8s.io/core:/stable:/v1.30/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
sudo systemctl restart jenkins

```

---

### B. Kubernetes Cluster Setup

#### Prerequisites (Execute on BOTH `k8s-master` and `k8s-worker`)

```bash
# 1. Disable swap completely
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Kernel modules & sysctl configuration
cat <<EOF # --system -p -y /etc/containerd /etc/containerd/config.toml /etc/modules-load.d/k8s.conf /etc/sysctl.d/k8s.conf 3. <<EOF EOF Install and apt apt-transport-https br_netfilter ca-certificates cat config configure containerd curl default gnupg install mkdir modprobe net.bridge.bridge-nf-call-ip6tables="1" net.bridge.bridge-nf-call-iptables="1" net.ipv4.ip_forward="1" overlay runtime sudo sysctl tee update |>/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# 4. Install kubeadm, kubelet, and kubectl
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.30/deb/](https://pkgs.k8s.io/core:/stable:/v1.30/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

#### Control Plane Initialization (Run ONLY on `k8s-master`)

```bash
# 1. Initialize cluster control plane
sudo kubeadm init --apiserver-advertise-address=10.0.1.230 --pod-network-cidr=192.168.0.0/16

# 2. Set up local kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 3. Install Calico CNI network plugin
kubectl apply -f [https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml)

# 4. Print worker join token
kubeadm token create --print-join-command

```

#### Join Worker Node (Run ONLY on `k8s-worker`)

```bash
sudo kubeadm join 10.0.1.230:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```

---

### C. Connect Jenkins to Kubernetes Cluster

1. On `k8s-master`, display the admin configuration:
```bash
cat ~/.kube/config

```


2. On `jenkins-server`, paste the configuration for the `jenkins` user:
```bash
sudo mkdir -p /var/lib/jenkins/.kube
sudo nano /var/lib/jenkins/.kube/config
# Paste credentials, verify server is [https://10.0.1.230:6443](https://10.0.1.230:6443)
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube

```


3. Test connectivity on `jenkins-server`:
```bash
sudo -u jenkins kubectl get nodes

```



---

## 5. Repository File Structure

```text
devops-static-app/
├── index.html              # Custom glassmorphic landing page
├── styles.css              # Application styling
├── Dockerfile              # Multi-stage lightweight Nginx container
├── k8s-deployment.yaml     # Kubernetes Deployment (2 replicas)
├── k8s-service.yaml        # NodePort Service (Port 32000)
├── Jenkinsfile             # Declarative CI/CD pipeline
└── README.md               # Infrastructure and workflow documentation

```

---

## 6. Manifest & Configuration Files

### `Dockerfile`

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

```

### `k8s-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-web-deployment
  labels:
    app: static-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-web
  template:
    metadata:
      labels:
        app: static-web
    spec:
      containers:
      - name: static-web-container
        image: <YOUR_DOCKERHUB_USERNAME>/devops-static-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80

```

### `k8s-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: static-web-service
spec:
  type: NodePort
  selector:
    app: static-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32000

```

### `Jenkinsfile`

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = '<YOUR_DOCKERHUB_USERNAME>'
        IMAGE_NAME         = 'devops-static-app'
        DOCKER_CREDS_ID    = 'dockerhub-creds'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} .'
                sh 'docker tag ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest'
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}'
                    sh 'docker push ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest'
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                sh 'sed -i "s|image:.*|image: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}|g" k8s-deployment.yaml'
                sh 'kubectl apply -f k8s-deployment.yaml'
                sh 'kubectl apply -f k8s-service.yaml'
                sh 'kubectl rollout status deployment/static-web-deployment'
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}

```

---

## 7. Verification & Monitoring Commands

Verify cluster and deployment status on `k8s-master`:

```bash
# Check node readiness
kubectl get nodes -o wide

# Check pod distribution across nodes
kubectl get pods -o wide -l app=static-web

# Inspect the NodePort service
kubectl get svc static-web-service

# Check rollout status
kubectl rollout status deployment/static-web-deployment

```

### Access Application

Open any web browser and navigate to:

```text
[http://154.21.235.152:32000](http://154.21.235.152:32000)

```

```

```
