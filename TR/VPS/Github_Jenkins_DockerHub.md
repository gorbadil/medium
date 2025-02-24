# Github, Jenkins ve DockerHub ile Otomasyon

Bu yazıda Github, Jenkins ve DockerHub arasında otomasyon yaparak, Github'da yapılan her commit sonrası Jenkins'in otomatik olarak build yapmasını ve DockerHub'a push etmesini sağlayacağız. Bu sayede, DockerHub'da oluşturduğumuz imajı her commit sonrası otomatik olarak güncelleyebileceğiz.

Jenkins Kubernetes üzerinde master-slave halinde çalışıyor. Jenkins master, Jenkins slave'ler üzerinden Docker imajlarını oluşturacak. Jenkins slave'ler üzerinde Docker yüklü olması gerekiyor. Jenkins slave'ler üzerinde Docker yüklü olması için aşağıdaki komutları çalıştırabilirsiniz.

[Jenkins Kubernetes Üzerinde](https://medium.com/@gorbadil/jenkins-on-kubernetes-8d2c422c08b8) yazısını inceleyebilirsiniz.

## VPS

Jenkins slave'ler için Dockerfile düzenlememiz gerekiyor.

Jenkins Master Dockerfile

```Dockerfile
FROM jenkins/jenkins:lts-slim-jdk17
USER root
RUN apt update && curl -fsSL https://get.docker.com | sh
RUN usermod -aG systemd-journal jenkins
RUN jenkins-plugin-cli --plugins kubernetes
USER jenkins
```

Jenkins Slave Dockerfile

```Dockerfile
FROM jenkins/agent
USER root
RUN apt update && curl -fsSL https://get.docker.com | sh
RUN usermod -aG systemd-journal jenkins
USER jenkins
```

- systemd-journal grubuna jenkins kullanıcısını eklememizin sebebi, Jenkins'in Docker komutlarını çalıştırabilmesi için gereklidir. Eğer aşağıdaki komut çıktısında group yetkisi farklı bir grup ise, Docker komutlarını çalıştırırken hata alabilirsiniz.

```bash
$ ls -l /var/run/docker.sock
srw-rw---- 1 root systemd-journal 0 Mar  1 14:00 /var/run/docker.sock
```

```bash
docker build -t <dockerhub kullanıcı adı>/jenkins-master -f Dockerfile .
docker build -t <dockerhub kullanıcı adı>/jenkins-slave -f Dockerfile .
```

- Eğer systemd-journal grubu farklı bir grup ise, Dockerfile içinde bulunan systemd-journal alanını değiştirmeniz gerekmektedir.

Jenkins slave'lerin Dockerfile'larını oluşturduktan sonra, Jenkins slave'lerin imajlarını oluşturup Kubernetes üzerinde çalıştırıyoruz. Jenkins slave'lerin Kubernetes üzerinde çalıştırılması için aşağıdaki komutları çalıştırabilirsiniz.

Yukarıdaki yazıda bulunan deployment.yaml dosyasında image kısmını, bu yeni oluşturduğumuz imaj ile değiştiriyoruz.

```yaml
containers:
  - name: jenkins-master
    image: <dockerhub kullanıcı adı>/jenkins-master
    imagePullPolicy: Always
```

## Github

Öncelikle Github'da bir repository oluşturuyoruz. Bu repository'e Jenkins'in erişimi olacak. Bu repository'e bir Dockerfile ekleyerek, Jenkins'in bu Dockerfile'ı kullanarak imaj oluşturmasını sağlayacağız. Projeyi ve Dockerfile dosyasını basit tutmak adına bir React uygulaması oluşturuyoruz.

```bash
npm create vite@latest react-app -- --template react
```

Bu komut ile React uygulaması oluşturulur. Ardından, oluşturulan uygulamanın içine bir Dockerfile ekliyoruz.

```Dockerfile
# build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json .
# install dependencies
RUN npm install
# copy files
COPY . .
# build app
RUN npm run build
# nginx stage
FROM nginx:stable-alpine

# Copy config nginx
COPY --from=build /app/.nginx/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Dockerfile içerisinde bulunan .nginx klasörü altında nginx.conf dosyası oluşturulur. Bu dosya içerisinde nginx konfigürasyonları bulunur. Bu konfigürasyonun asıl amacı, React uygulamasında Router kullanıldığında, sayfaların yeniden yüklenmesi durumunda 404 hatası alınmamasını sağlamaktır. Eklediğimiz uygulama için gerekli bir durum değil ama kim bilir, ileride ihtiyacımız olabilir.

```nginx
server {

  listen 80;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri /index.html =404;
  }

  error_page   500 502 503 504  /50x.html;

  location = /50x.html {
    root   /usr/share/nginx/html;
  }
}
```

Bu dosyaları ekledikten sonra, projeyi Github'a push ediyoruz. Ardından Github Webhook oluşturarak Jenkins ile bağlantı kuracağız. Bu yazıda bu konuya yeniden değinmeyeceğim. Github Webhook oluşturmak için [bu yazıyı](https://medium.com/@gorbadil/github-webhook-ile-jenkins-otomasyonu-b9285a4322f2) inceleyebilirsiniz.

## Jenkins

Öncelikle pipeline kullanabilmek için Jenkins içerisinde Docker Pipeline ve Docker plugin'lerini yüklememiz gerekiyor. Jenkins ana sayfasında `Manage Jenkins` butonuna tıklıyoruz. Ardından, `Plugins` butonuna tıklıyoruz. Açılan sayfada, `Available` sekmesine tıklıyoruz. Burada, `Docker Pipeline` ve `Docker` plugin'lerini aratıp yüklüyoruz.

Docker plugin'lerini yükledikten sonra, Kubernetes cloud alanında kendi image'imizi kullanabilmemiz için oluşturduğumuz Kubernetes Cloud için Pod Template hazırlıyoruz.

Jenkins Ana Sayfa -> Manage Jenkins -> Configure System -> Cloud -> Kubernetes -> Add Pod Template

Burada Name, Namespace ve Label alanlarını hazırladıktan sonra Add Container diyerek kendi image'imizi ekliyoruz.

![Container Template](../images/Container_Template.png)

Ardından Advanced alanına geçiyoruz. Burada Run in privileged container seçeneğini işaretliyoruz. Bu seçenek, Jenkins'in Docker komutlarını çalıştırabilmesi için gereklidir.

![Run in Privileged](../images/RuninPrivileged.png)

Derdimiz henüz son bulmadı. Bulması da beklenemezdi.

Volume alanına geçiyoruz. Burada HostPath Type seçeneğini seçiyoruz. HostPath alanına /var/run/docker.sock yazıyoruz. Bu alan, Jenkins'in Docker komutlarını çalıştırabilmesi için gereklidir.

![Container Volume](../images/Container_Volumes.png)

Artık Jenkins slave'lerimizde Docker komutlarını çalıştırabiliriz.

Docker Hub üzerinde işlem yapabilmek için Docker Hub credentials oluşturuyoruz.

Jenkins Ana Sayfa -> Manage Jenkins -> Credentials -> System -> Global credentials -> Add Credentials

Açılan sayfada, `Kind` kısmından `Username with password` seçeneğini seçiyoruz. `Username` kısmına DockerHub kullanıcı adınızı, `Password` kısmına DockerHub şifrenizi giriyoruz. `ID` kısmına `docker-hub-credentials` yazıyoruz. Ardından, `OK` butonuna tıklıyoruz.

Jenkins ana sayfasına dönüyoruz ve `New Item` butonuna tıklıyoruz. Açılan sayfada, işimizin adını giriyoruz ve `Pipeline` seçeneğini seçiyoruz. Açılan sayfada, Github'dan alacağımız projenin URL'sini ekliyoruz. Build Trigger kısmında, `GitHub hook trigger for GITScm polling` seçeneğini seçiyoruz. Bu seçenek, Github'da yapılan her commit sonrası Jenkins'in otomatik olarak build yapmasını sağlar. Ardından, `Pipeline` kısmına geçiyoruz.

Pipeline Script seçili iken aşağıdaki kodları ekliyoruz.

```groovy
pipeline {
    agent {
        kubernetes {
            label 'jenkins-slave'
        }
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: '<Github repository URL>'
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                container('jenkins-slave') {
                    script {
                        docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                            def customImage = docker.build("<DockerHub kullanıcı adı>/<DockerHub repository adı>:latest")
                            customImage.push()
                        }
                    }
                }
            }
        }
    }
}

```

Bu kodlar, Jenkins'in Github'dan alacağı projeyi Jenkins slave üzerinde build edip, DockerHub'a push etmesini sağlar. Kodlar içerisinde bulunan `<Github repository URL>`, `<DockerHub kullanıcı adı>` ve `<DockerHub repository adı>` kısımlarını kendi bilgileriniz ile değiştirmeniz gerekmektedir.

Artık Jenkins'in Github'dan alacağı projeyi Jenkins slave üzerinde build edip, DockerHub'a push etmesini sağlayabiliriz. Bu sayede, Github'da yapılan her commit sonrası DockerHub'da oluşturduğumuz imajı otomatik olarak güncelleyebiliriz.

## Sonuç

Bu yazıda Github, Jenkins ve DockerHub arasında otomasyon yaparak, Github'da yapılan her commit sonrası Jenkins'in otomatik olarak build yapmasını ve DockerHub'a push etmesini sağladık. Bu sayede, DockerHub'da oluşturduğumuz imajı her commit sonrası otomatik olarak güncelleyebileceğiz.
