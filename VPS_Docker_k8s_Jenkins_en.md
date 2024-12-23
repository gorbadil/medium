# VPS Docker Kubernetes Jenkins Installation and Nginx Reverse Proxy

In this article, I will explain how to install a ready environment for CI/CD processes with Jenkins on VPS. By running the Jenkins service on Kubernetes, we will ensure the scalability of Jenkins.

It is not a preferred automation method to perform operations on the Jenkins master. We will run Jenkins master and agents on Kubernetes and ensure the scalability of Jenkins.

First, we will install Docker and Kubernetes on the system. Then we will run the Jenkins service on Kubernetes.

I will use Ubuntu operating system on VPS.

## VPS Installation

In the previous article, I explained how to install a secure VPS server. You can access the VPS Installation from the link below.

- [VPS Installation](https://medium.com/@gorbadil/how-to-setup-a-secure-vps-server-7597b95440c9)

## Docker Installation

Docker is a popular tool that allows us to containerize applications. We can install Docker on our VPS server by following the steps below.

First, let's add the Docker repository.

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Then we can install Docker.

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

We can test the Docker installation by running the following command.

```bash
sudo docker run hello-world
```

If we want to remove the sudo command from the Docker commands.

```bash
sudo usermod -aG docker $USER
```

We need to log out and log in again for the changes to take effect.

Docker installation link should also be here, it may be needed.

- [Docker Installation](https://docs.docker.com/engine/install/ubuntu/)

## Kubernetes Installation

Kubernetes is a popular tool used for container orchestration. By running the Jenkins service on Kubernetes, we will ensure the scalability of Jenkins. We can install Kubernetes on our VPS server by following the steps below.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

We can test Kubernetes by running the following command.

```bash
kubectl cluster-info
```

Kubernetes installation link should also be here, it may be needed.

- [Kubernetes Installation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Jenkins

Now let's move on to the installation of the Jenkins service that we will use for automation.

Jenkins is a popular tool used for automation. We can install Jenkins on our VPS server by following the steps below.

- [Jenkins Official Website](https://www.jenkins.io/)
- [Jenkins Docker Image](https://hub.docker.com/r/jenkins/jenkins)
- [Jenkins Kubernetes Installation](https://www.jenkins.io/doc/book/installing/kubernetes/)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)
- [Jenkins Kubernetes Plugin Configuration](https://www.jenkins.io/doc/book/pipeline/kubernetes/)

Let's install Jenkins on Kubernetes. Let's not confuse when we see the links. When installing something, it is best to look at the official site of that thing. Installation steps and documents are available on the official site. I will try to explain step by step.

First, let's install Jenkins Master on Kubernetes. Let's follow the steps below.

1. Create Namespace
2. Create Service Account
3. Create Volume
4. Create Jenkins Master Deployment
5. Create Jenkins Master Service

You can access all files
[here](git clone https://github.com/scriptcamp/kubernetes-jenkins) on GitHub.

### Namespace Creation

We can use namespaces to make it easier to manage resources on Kubernetes. Then let's use it. Let's create a namespace called devops-tools for services that we can run on Kubernetes in the future.

```bash
kubectl create namespace devops-tools
```

### Service Account Creation

To run Jenkins Master on Kubernetes, we need to create a service account. We can use the following yaml file to create a service account.

```bash
vim serviceAccount.yaml
```

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
  - kind: ServiceAccount
    name: jenkins-admin
    namespace: devops-tools
```

Let's remember how we were getting out of the vim editor for those who don't remember.

First, press the ESC key. Then type the following command.

```bash
:wq
```

Let's create a service account using the yaml file above.

```bash
kubectl apply -f serviceAccount.yaml
```

### Volume Creation

To run Jenkins Master on Kubernetes, we need to create a volume. We can use the following yaml file to create a volume.

```bash
vim volume.yaml
```

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage
  claimRef:
    name: jenkins-pv-claim
    namespace: devops-tools
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - minikube
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

In the yaml file above, we created a volume of 10Gi. We connected the volume to the /mnt directory on minikube because we are working with minikube.

If you are going to use a different worker, you need to change the path in the volume file. You can use the following command to learn this.

```bash
kubectl get nodes
```

Let's create a volume using the yaml file above.

```bash
kubectl apply -f volume.yaml
```

### Jenkins Master Deployment Creation

To run Jenkins Master on Kubernetes, we need to create a deployment. We can use the following yaml file to create a deployment.

```bash
vim deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
            claimName: jenkins-pv-claim
```

Let's explain what we used in this file.

1. We give access permissions to the Jenkins pod's volume with "securityContext".
2. We monitor the Jenkins pod with "livenessProbe" and "readinessProbe".
3. We connect the /var/jenkins_home directory on the Jenkins pod to the volume so that the Jenkins pod's data is not lost.

Let's create a deployment using the yaml file above.

```bash
kubectl apply -f deployment.yaml
```

## Test Installation

Congratulations! You have successfully run Jenkins Master on Kubernetes. Now let's check if Jenkins Master is running.

```bash
kubectl get deployments -n devops-tools
```

We can use the following command to access details about the Jenkins Master Deployment.

```bash
kubectl describe deployments --namespace=devops-tools
```

## Access

Great! We installed Jenkins Master, but we can't access it from the outside world. Oh no. Let's delete it then.

Bad joke. Let's make Jenkins Master accessible from the outside world.

```bash
vim service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: /
    prometheus.io/port: "8080"
spec:
  selector:
    app: jenkins-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32000
```

Let's create a Jenkins Master service using the yaml file above.

```bash
kubectl apply -f service.yaml
```

## Nginx Installation

Great! Now we can access Jenkins Master from the outside world. We need to use a reverse proxy because we are working on VPS. I will use Nginx. You can use Apache, Caddy, or something else.

```bash
sudo apt-get install nginx
```

After installing Nginx, let's start the Nginx service and make it start automatically at startup.

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Let's check

```bash
sudo systemctl status nginx
```

Let's add Nginx to the firewall settings.

```bash
sudo ufw allow 'Nginx Full'
```

Let's check the firewall status.

```bash
sudo ufw status
```

Let's create an Nginx configuration file.

```bash
vim /etc/nginx/conf.d/jenkins.conf
```

```nginx
server {
    listen 80;
    server_name jenkins.example.com;

    location / {
        proxy_pass http://localhost:32000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Don't forget to replace jenkins.example.com with your own domain

Let's create the Nginx configuration using the configuration file above.

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Now we can access Jenkins Master via the domain. You can go to the following address to log in.

```
http://jenkins.example.com
```

## Jenkins Administator Password

To get the Jenkins Administator password, first get the pod name.

```bash
kubectl get pods --namespace=devops-tools
```

Then use the following command to get the Jenkins Administator password.

```bash
kubectl logs <pod name> --namespace=devops-tools
```

## Conclusion

We have successfully installed Jenkins on Kubernetes. We have made Jenkins Master accessible from the outside world. We have made Jenkins Master accessible via Nginx reverse proxy. We have successfully completed the installation.
