# VPS Üzerinde Continuous Integration/Continuous Deployment (CI/CD) Kurulumu ve Nginx Reverse Proxy

Bu yazıda, VPS üzerinde Jenkins ile CI/CD süreçlerine hazır bir ortam kurulumunu anlatacağım. Jenkins servisini Kubernetes üzerinde çalıştırarak, Jenkins'in ölçeklendirilmesini sağlayacağız.

Jenkins master üzerinde işlem yapmak tercih edilen bir otomasyon yöntemi değildir. Bizde Kubernetes üzerinde Jenkins master ve agent'lerini çalıştırarak, Jenkins'in ölçeklendirilmesini sağlayacağız.

Sırasıyla sisteme öncelikle Docker ve Kubernetes kurulumunu gerçekleştireceğiz. Daha sonra Jenkins servisini Kubernetes üzerinde çalıştıracağız.

VPS üzerinde Ubuntu işletim sistemi kullanacağım.

Bir önceki yazımda anlattığım VPS Kuruluma aşağıdaki linkten ulaşabilirsiniz.

- [VPS Sunucusu Kurulumu](https://medium.com/@gorbadil/güvenli-bir-vps-sunucusu-nasıl-kurulur-daca9dec1ca0)

## Docker Kurulumu

Docker, uygulamaları konteynerleştirmemizi sağlayan popüler bir araçtır. Docker'ı VPS sunucumuza aşağıdaki adımları takip ederek kurabiliriz.

Öncelikle Docker reposunu ekleyelim.

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

Ardından Docker kurlumunu gerçekleştirebiliriz.

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Kurulumu test etmek için aşağıdaki komutu çalıştırabiliriz.

```bash
sudo docker run hello-world
```

Docker komutlarından sudo komutunu kaldırmak istersek.

```bash
sudo usermod -aG docker $USER
```

Docker kurulum linki de burada bulunsun lazım olur filan.

- [Docker Kurulumu](https://docs.docker.com/engine/install/ubuntu/)

## Kubernetes Kurulumu

Kubernetes, konteyner orkestrasyonu için kullanılan popüler bir araçtır. Jenkins servisini Kubernetes üzerinde çalıştırarak, Jenkins'in ölçeklendirilmesini sağlayacağız. Kubernetes'i VPS sunucumuza aşağıdaki adımları takip ederek kurabiliriz.

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

Kubernetes'i test etmek için aşağıdaki komutu çalıştırabiliriz.

```bash
kubectl cluster-info
```

Kurulum linki de burada bulunsun lazım olur filan.

- [Kubernetes Kurulumu](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Jenkins

Geldik otomasyon için kullanacak olduğumuz Jenkins servisinin kurulumuna.

- [Jenkins Resmi Sitesi](https://www.jenkins.io/)
- [Jenkins Docker Image](https://hub.docker.com/r/jenkins/jenkins)
- [Jenkins Kubernetes Installation](https://www.jenkins.io/doc/book/installing/kubernetes/)
- [Jenkins Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)
- [Jenkins Kubernetes Plugin Configuration](https://www.jenkins.io/doc/book/pipeline/kubernetes/)

Linkleri görünce kafalar karışmasın. Herhangi bir şeyi kurarken, o şeyin resmi sitesine bakmak en doğrusudur. Resmi sitesinde kurulum adımları ve belgeleri bulunmaktadır. Yine de adım adım anlatmaya çalışacağım.

Öncelikle Jenkins Master'ı Kubernetes üzerinde kuralım. Sırasıyla aşağıdaki adımları takip edelim.

1. Namespace Oluşturma
2. Service Account Oluşturma
3. Volume Oluşturma
4. Jenkins Master Deployment Oluşturma
5. Jenkins Master Service Oluşturma

Tüm dosyalara [buradan](git clone https://github.com/scriptcamp/kubernetes-jenkins) ulaşabilirsiniz.

### Namespace Oluşturma

Kubernetes üzerindeki kaynakları yönetmeyi kolaylaştırmak için namespace'ler kullanabiliriz. O zaman kullanalım. Kubernetes üzerinde ileride de çalıştırabileceğimiz servisler için devops-tools adında bir namespace oluşturalım.

```bash
kubectl create namespace devops-tools
```

### Service Account Oluşturma

Jenkins Master'ın Kubernetes üzerinde çalışabilmesi için bir service account oluşturmamız gerekmektedir. Service account oluşturmak için aşağıdaki yaml dosyasını kullanabiliriz.

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

Hatırlamayanlar için vim editörden nasıl çıkıyorduk hatırlayalım.

Önce ESC tuşuna basıyoruz. Sonra aşağıdaki komutu yazıyoruz.

```bash
:wq
```

Yukarıdaki yaml dosyasını kullanarak service account oluşturalım.

```bash
kubectl apply -f serviceAccount.yaml
```

### Volume Oluşturma

Jenkins Master'ın çalışabilmesi için bir volume oluşturmamız gerekmektedir. Volume oluşturmak için aşağıdaki yaml dosyasını kullanabiliriz.

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

Burada volume dosyasını minikube ile çalıştığımız için minikube üzerindeki /mnt dizinine bağladık.

Eğer farklı bir worker kullanacaksanız, volume dosyasındaki path kısmını değiştirmeniz gerekmektedir. Bunu da öğrenmek için aşağıdaki komutu kullanabilirsiniz.

```bash
kubectl get nodes
```

Yukarıdaki yaml dosyasını kullanarak volume oluşturalım.

```bash
kubectl apply -f volume.yaml
```

### Jenkins Master Deployment Oluşturma

Jenkins Master'ı Kubernetes üzerinde çalışabilmesi için bir deployment oluşturmamız gerekmektedir. Deployment oluşturmak için aşağıdaki yaml dosyasını kullanabiliriz.

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

Bu dosyada kullandıklarımızı açıklamak gerekirse;

1. "securityContext" ile Jenkins pod'unun volume'üne erişim izinlerini veriyoruz.
2. Jenkins pod'unu izleyebilmek için "livenessProbe" ve "readinessProbe" tanımlıyoruz.
3. Jenkins pod üzerindeki /var/jenkins_home dizinine volume bağlıyoruz böylelikle Jenkins pod'unun verileri kaybolmaz.

Yukarıdaki yaml dosyasını kullanarak Jenkins Master deployment oluşturalım.

```bash
kubectl apply -f deployment.yaml
```

## Kurulum Testi

Tebrikler! Jenkins Master'ı Kubernetes üzerinde başarıyla çalıştırdınız. Şimdi Jenkins Master'ın çalışıp çalışmadığını kontrol edelim.

```bash
kubectl get deployments -n devops-tools
```

Deployment hakkında detaylara ulaşmak için aşağıdaki komutu kullanabiliriz.

```bash
kubectl describe deployments --namespace=devops-tools
```

## Erişim

Harika! Jenkins Master kurduk ancak dış düşyadan erişemiyoruz. Hayda. Silelim o zaman.

Tamam kötü şaka. Hadi Jenkins Master'ı dış dünyadan erişilebilir hale getirelim.

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

Yukarıdaki yaml dosyasını kullanarak Jenkins Master service oluşturalım.

```bash
kubectl apply -f service.yaml
```

## Nginx Kurulumu

Çok güzel artık VPS üzerinden Jenkins Master'a erişebiliriz. Jenkins Master'a erişmek için aşağıdaki komutu kullanabiliriz. Yine de elbette bitmedi.

VPS üzerinde çalıştığımız için bir reverse proxy kullanmamız gerekmektedir. Ben Nginx kullanacağım. Siz Apache, Caddy veya başka bir şey kullanabilirsiniz.

```bash
sudo apt-get install nginx
```

Nginx kurulumunu gerçekleştirdikten sonra aşağıdaki komutu kullanarak Nginx servisini başlatalım ve başlangıçta otomatik olarak başlamasını sağlayalım.

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Nginx konfigürasyon dosyasını oluşturalım.

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

```
jenkins.example.com yerine kendi domaininizi yazmayı unutmayın.
```

Yukarıdaki konfigürasyon dosyasını kullanarak Nginx konfigürasyonunu oluşturalım.

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Artık domain üzerinden Jenkins Master'a erişebiliriz. Giriş yapmak için aşağıdaki adrese gidebiliriz.

```
http://jenkins.example.com
```

Administator şifresini almak için sırasıyla önce pod ismini alalım.

```bash
kubectl get pods --namespace=devops-tools
```

Sonra pod ismini kullanarak aşağıdaki komutu çalıştıralım.

```bash
kubectl logs <pod name> --namespace=devops-tools
```

## Sonuç

Artık Jenkins Master'ı Kubernetes üzerinde çalıştırarak, Jenkins'in ölçeklendirilmesini sağladık. Jenkins Master'ı dış dünyadan erişilebilir hale getirdik. Jenkins Master'ı Nginx üzerinden reverse proxy kullanarak erişilebilir hale getirdik.
