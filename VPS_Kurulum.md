# Güvenli Bir VPS Sunucusu Nasıl Kurulur

Selam arkadaşlar, bugün güvenli bir VPS sunucusu nasıl kurulur, adım adım anlatacağım.

## "VPS Sunucusu Neden Kuralım, Ne İçin Kullanalım?"

### VPN

Evet, VPS ile kendi VPN’imizi kurabiliriz. Elbette "erişimin yasaklı olduğu sitelere ulaşın" demiyorum. Sadece yapabiliyoruz.

### Web Sunucusu

Geliştirdiğimiz projeleri kendi sunucumuzda çalıştırmak için. Çünkü neden yapmayalım?

### Bulut Depo

Evet, ücretsiz birçok çözüm var, biliyorum. Sonuçta burası bizim. Neden burayı kullanmayalım ki?

### Pratik

Evet, container teknolojilerini duyuyoruz, öğreniyoruz. Ancak bunu gerçekten bir sunucu üzerinde çalıştırmak, sonuç almak... Neden yapmayalım ki?

### Eğlence

Tüm bunları yapabiliyorken neden yapmayalım ki?

---

## VPS Alternatifleri

Peki, neden VPS kullanalım ya da VPS dışında ne seçeneklerimiz olabilir? Cloud Service (Bulut Hizmetleri) seçenekleri şimdilik gündemimizde değil. İlerleyen zamanlarda AWS, Azure, GCP özelinde de konuşuruz. Ancak sunucu için seçeneklerimiz şunlardır:

### Fiziksel Sunucu

Adından da anlayabileceğimiz gibi, komple bir sunucu kiralamaktan bahsediyoruz. Bu dolar kurunda bunu yapabiliyorsak ne güzel.

### Sanal Sunucu

Fiziksel bir sunucunun sanallaştırma teknolojileri ile ayrılmış bölümleri üzerinde çalışan sunucu. Biz tam burada, bu alandayız.

**VPS ve VDS arasındaki fark:**

- **VPS:** Yazılımsal olarak ayrılmıştır.
- **VDS:** Donanımsal olarak ayrılmıştır.

#### Neden VDS Seçmiyoruz?

Çünkü pahalı.

#### Shared Sunucular

Çalışan bir işletim sistemi üzerinde bize verilen bir kullanıcı hesabı. Root erişim olmadığından kullanacağımız teknolojiler için uygun değil.

---

Neyse ya, iyice uzattık. Haydi başlayalım!

# VPS Kurulum

## VPS Sağlayıcı

VPS sağlayıcı seçimi önemli. Doğru sanallaştırma teknolojisi kullanımı, destek, kullanıcı görüşleri gibi konular ileride başımızı ağrıtmaması için önemli. Gidin şuradan kiralayın demeyeceğim ancak adı sanı duyulmamış yerlerde gezmenizi önermem. Fiyatı uygun, bilindik, kullanıcı görüşleri olumlu bir sağlayıcıdan yolumuza devam.

## VPS Sunucu Bağlantı

Kullanacağımız VPS sunucusuna SSH üzerinden bağlanarak hayatımıza başlıyoruz. VPS Sağlayıcı bizim için oluşturduğu root kullanıcısı ve parolası ile hayatımıza başlıyoruz.

İşletim sistemi olarak Linux/Ubuntu tercih ederek devam ediyorum.

Sunucumuza bağlanalım

```bash
ssh root@sunucu_ip_adresi
```

## Güvenlik

Sunucumuza bağlandık, ilk işimiz güvenliği sağlamak. Öncelikle kendi kullanıcımızı oluşturalım ve sudo yetkisi verelim.

```bash
adduser kullanici_adi
usermod -aG sudo kullanici_adi
```

Password girişini kullanmak yerine SSH anahtarını kullanarak giriş yapmak sunucu güvenliğimizi arttırır. Bu sebeple host makinemizde (kendi bilgisayarımız) SSH anahtar çifti oluşturuyoruz. Windows için [Putty](https://www.putty.org/) kullanabiliriz. Linux/MacOS için terminal üzerinden aşağıdaki komutu çalıştırabiliriz.

```bash
ssh-keygen -t rsa -b 4096
```

Özellikle bir dosya yolu belirtmediysek anahtarlarımız `~/.ssh` klasörüne oluşturulur. Public anahtarımızı sunucumuza gönderelim.

```bash
ssh-copy-id ~/.ssh/id_rsa.pub kullanici_adi@sunucu_ip_adresi
```

- Eğer sistemde ssh-copy-id komutu yoksa, aşağıdaki komutu çalıştırabiliriz.

```bash
cat ~/.ssh/id_rsa.pub | ssh kullanici_adi@sunucu_ip_adresi "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Sunucu üzerinde ssh servisini yeniden başlatarak anahtar tabanlı giriş yapabiliriz.

```bash
sudo service ssh restart
```

Test edelim.

```bash
ssh kullanici_adi@sunucu_ip_adresi
```

Harika. Şimdi password ile girişi de kapattığımızda sunucumuz daha güvenli olacaktır.

```bash
sudo nano /etc/ssh/sshd_config
```

Dosya içerisinde `PasswordAuthentication` satırını bulup `no` yapalım.

```bash
PasswordAuthentication no
```

Değişiklikleri kaydedip servisi yeniden başlatalım.

- nano'ya yabancı kullanıcılar için kayıt etme işlemi `Ctrl + O` tuşlarına basıp `Enter` tuşuna basarak yapılır. Çıkış işlemi ise `Ctrl + X` tuşlarına basarak yapılır.

```bash
sudo service ssh restart
```

## Güncellemeler

Sunucumuzun güncel olması önemli. Güvenlik açıklarının kapatılması, yeni özelliklerin eklenmesi için güncellemeleri yapalım.

```bash
sudo apt update && sudo apt upgrade -y
```

## Firewall

Sunucumuzun güvenliği için firewall kurulumu yapalım. `ufw` paketi ile firewall kurulumu yapabiliriz.

- Burada paketi çalıştırmadan önce ssh servisine izin vermemiz gerekiyor. Aksi takdirde bağlantı kopar.

```bash
sudo apt install ufw
```

Varsayılan olarak gelen kuralları inceleyelim.

```bash
sudo ufw app list
```

SSH servisine izin verelim.

```bash
sudo ufw allow OpenSSH
```

Firewall'u aktif hale getirelim.

```bash
sudo ufw enable
```

Durumu kontrol edelim.

```bash
sudo ufw status
```

Çok sevdiğimiz paketleri kurarak başlayalım.

```bash
sudo apt install htop curl wget git vim -y
```

## OpenVPN Kurulumu

Sunucumuzda VPN servisi kurarak güvenli bir bağlantı sağlayabiliriz. OpenVPN paketini aşağıdaki adreste bulunan adımları takip ederek kurabiliriz.

[OpenVPN Kurulumu](https://openvpn.net/as-docs/ubuntu.html#invalid-certificate-76593)

## Web Sunucusu Kurulumu

Sunucumuzda web sunucusu olarak Nginx kullanacağız. Nginx paketini aşağıdaki adreste bulunan adımları takip ederek kurabiliriz.

```bash
sudo apt install nginx
```

Firewall ayarlarını gerçekleştirelim.

```bash
sudo ufw allow 'Nginx Full'
```

Firewall durumunu kontrol edelim. Burada izin verilenler arasında Nginx Full olmalı.

```bash
sudo ufw status
```

Sunucu durumunu kontrol edelim.

```bash
sudo systemctl status nginx
```

Çıktı olarak `active (running)` almalıyız.

```bash
Output
● nginx.service - A high performance web server and a reverse proxy server
    plaintext
        Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
        Active: active (running) since Fri 2020-04-20 16:08:19 UTC; 3 days ago
          Docs: man:nginx(8)
     Main PID: 2369 (nginx)
         Tasks: 2 (limit: 1153)
        Memory: 3.5M
        CGroup: /system.slice/nginx.service
                  ├─2369 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
                  └─2380 nginx: worker process
```

Tarayıcımızdan sunucumuzun IP adresine gittiğimizde Nginx sayfasını görmeliyiz.

nginx kontrol etmek için komutlar

```bash
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl disable nginx
sudo systemctl enable nginx
```

Nginx servisinin active ve enable olduğuna emin olalım.

## SSL Sertifikası

Sunucumuzda SSL sertifikası kullanarak güvenli bir bağlantı sağlayabiliriz. Certbot paketini aşağıdaki adreste bulunan adımları takip ederek kurabiliriz.
[Certbot Kurulumu](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx)
Kurulum sonrasında sertifikayı oluşturalım.

```ash
sudo certbot --nginx
```

## İlk Websitemiz

Sunucumuzda web sitesi yayınlamak için `/var/www/html` klasörüne dosyalarımızı yükleyebiliriz. Örnek bir index.html dosyası oluşturalım.

```bash
sudo vim /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Bu Bir Web Sitesidir</title>
  </head>
  <body>
    <h1>Hoş geldin kardeşim</h1>
  </body>
</html>
```

Tarayıcımızdan sunucumuzun IP adresine gittiğimizde oluşturduğumuz web sitesini görmeliyiz.

## Sonuç

Evet, güvenli bir VPS sunucusu nasıl kurulur, adım adım anlattım. Umarım faydalı olmuştur. Sorularınızı yorumlarda belirtebilirsiniz. İyi çalışmalar.
