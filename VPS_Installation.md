# How to Setup a Secure VPS Server

Hello friends, today I will explain step by step how to set up a secure VPS server.

## "Why Should We Set Up a VPS Server, What Should We Use It For?"

### VPN

Yes, we can set up our own VPN with VPS. Of course, I'm not saying "access blocked sites". We can do it. But we can also use it for secure access to our own network.

### Web Server

We can run our own web server. Why not?

### Cloud Storage

Yes, there are free solutions, I know. After all, this is ours. Why don't we use it?

### Practice

Yes, we hear about container technologies, we learn. But running it on a real server, getting results... Why not?

### Fun

Yes, we can do all of this. Why not?

---

## VPS Alternatives

Okay, why should we use VPS or what other options do we have besides VPS? Cloud Service options are not on our agenda for now. We will talk about AWS, Azure, GCP in the future. However, our server options are as follows:

### Physical Server

As we can understand from the name, we are talking about renting a complete server. How nice if we can do this at this exchange rate.

### Virtual Server

A server that works on divided sections with virtualization technologies of a physical server. We are right here, in this area.

**VPS and VDS Difference:**

- **VPS:** Software is divided.
- **VDS:** Hardware is divided.

### Why Don't We Choose VDS?

Because it's expensive.

### Shared Servers

It is a server that is shared by many users. It is not suitable for us because it is not secure and we need root access.

---

Okay, we've stretched it out. Let's get started!

# VPS Installation

## VPS Provider

VPS provider selection is important. The use of the right virtualization technology, support, user reviews, etc. are important for not causing us headaches in the future. I won't say go rent from here, but I won't recommend wandering in places where the name is not heard. Go to a provider with a reasonable price, well-known, and positive user reviews.

## VPS Server Connection

We start our life by connecting to the VPS server we will use via SSH. The VPS provider starts our life with the root user and password it created for us.

I continue with Linux/Ubuntu as the operating system.

Let's connect to our server

```bash
ssh root@server_ip_address
```

## Security

We connected to our server, the first thing we do is to ensure security. First, let's create our own user and give sudo permission.

```bash
adduser username
usermod -aG sudo username
```

Password login is a security vulnerability. Using SSH key login instead of password login increases our server security. For this reason, we create an SSH key pair on our host machine (our computer). We can use [Putty](https://www.putty.org/) for Windows. For Linux/MacOS, we can run the following command in the terminal.

```bash
ssh-keygen -t rsa -b 4096
```

If we do not specify a file path, our keys are created in the `~/.ssh` folder. Let's send our public key to our server.

```bash
ssh-copy-id ~/.ssh/id_rsa.pub username@server_ip_address
```

- If the ssh-copy-id command is not available on the system, we can run the following command.

```bash
cat ~/.ssh/id_rsa.pub | ssh username@server_ip_address "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Restart the ssh service on the server to log in with the key-based login.

```bash
sudo service ssh restart
```

Let's test it.

```bash
ssh username@server_ip_address
```

If we can log in without a password, we have succeeded.

## SSH Configuration

Great! Now that we have logged in with the key, our server is more secure. Now that we have closed the login with the password, our server will be more secure.

```bash
sudo nano /etc/ssh/sshd_config
```

Let's find the `PasswordAuthentication` line in the file and make it `no`.

```bash
PasswordAuthentication no
```

Save the changes and restart the service.

```bash
sudo service ssh restart
```

## Updates

It is important that our server is up to date. Let's make updates to close security vulnerabilities and add new features.

```bash
sudo apt update && sudo apt upgrade -y
```

## Firewall

Let's install the firewall for the security of our server. We can install the firewall with the `ufw` package.

- We need to allow the ssh service before running the package. Otherwise, the connection will be disconnected.

```bash
sudo apt install ufw
```

Let's examine the default rules.

```bash
sudo ufw app list
```

Let's allow the SSH service.

```bash
sudo ufw allow OpenSSH
```

Let's activate the firewall.

```bash
sudo ufw enable
```

Let's check the status of the firewall.

```bash
sudo ufw status
```

Let's start by installing the packages we love.

```bash
sudo apt install htop curl wget git vim -y
```

## OpenVPN Installation

We can provide a secure connection by installing a VPN service on our server. We can install the OpenVPN package by following the steps at the address below.

[OpenVPN Installation](https://openvpn.net/as-docs/ubuntu.html#invalid-certificate-76593)

## Web Server Installation

We will use Nginx as the web server on our server. We can install the Nginx package by following the steps at the address below.

```bash
sudo apt install nginx
```

Let's check

```bash
sudo systemctl status nginx
```

Add nginx to the firewall settings.

```bash
sudo ufw allow 'Nginx Full'
```

Let's check the firewall status.

```bash
sudo ufw status
```

Let's check the server status.

```bash
sudo systemctl status nginx
```

As output, we should get `active (running)` as output.

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

When we go to the IP address of our server from our browser, we should see the Nginx page.

## Nginx Control Commands

```bash
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl disable nginx
sudo systemctl enable nginx
```

## SSL Certificate

We can provide a secure connection on our server using an SSL certificate. We can install the Certbot package by following the steps at the address below.
[Certbot Installation](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx)

After installation, let's create the certificate.

```bash
sudo certbot --nginx
```

## Our First Website

We can upload our files to `/var/www/html` folder to publish a website on our server. Let's create an example index.html file.

```bash
sudo vim /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>My First Website</title>
  </head>
  <body>
    <h1>Welcome to My First Website</h1>
  </body>
</html>
```

When we go to the IP address of our server from our browser, we should see the website we created.

## Result

Yes, I explained how to set up a secure VPS server step by step. I hope it was useful. You can specify your questions in the comments. Good work.
