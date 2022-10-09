# Multiple NextJS Apps on One Server
The goal is to server multiple NextJS apps that are running on different ports and each should be accessible from a different domain name.

# Install Nginx on Ubuntu on Raspberry Pi

Instead of paying for a server to play with, I set up nginx on a Raspberry Pi 4.

For the actual site deployment(s) that I will potentially do, I'll use a cloud server like a Digital Ocean droplet or EC2.

## Find Raspberry Pi IP Address:

You can get your Raspberry Pi ip address by connecting it to a display and logging in, then running `hostname -I`

e.g. `192.168.4.205`

```sh
# Make sure your ssh connection does not time out

sudo vi /etc/ssh/sshd_config
# Add this to the bottom of the file
IPQoS cs0 cs0
```

[Guide followed](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04).

```sh
sudo apt-get update
sudo apt-get install nginx
sudo ufw app list
```

It will list:

```sh
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

>  It is recommended that you enable the most restrictive profile that will still allow the traffic you’ve configured. Since we haven’t configured SSL for our server yet, in this guide, we will only need to allow traffic on port 80.

```sh
sudo ufw allow 'Nginx HTTP'

# This wasn't included in the guide.. but had to do it
sudo ufw allow 'OpenSSH'
sudo ufw enable

# verify it
sudo ufw status

# Check nginx running
systemctl status nginx
```

Visit the ip address of your raspberry pi in a web browser and you will see the "Welcome to nginx!" page.

# Practice - Configure 2 sites that demonstrate the concept: example.com and test.com

I followed [this](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04) excellent guide from Digital Ocean.

# How is it working?

1. When you hit example.com, the local host system redirects your request to the ip address you set in /etc/hosts.
2. Nginx sees the host name in the request and looks at the configured server blocks. It finds a match by looking at the configuration located at `/etc/nginx/sites-enabled/example.com`:

```nginx
server {
	listen 80;
	listen [::]:80;

	root /var/www/example.com/html;

	server_name example.com www.example.com;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}
}
```

# NextJS is a little different..

NextJS is not just html files. It does *have* static files but it's also a live Node application that we'll proxy with Nginx.

In other words, nginx will just pass along the request to the Next app and return the response to the client (except for `.next/static`)

I referred to this gist, https://gist.github.com/iam-hussain/2ecdb934a7362e979e3aa5a92b181153 for how to do this. It was extremely helpful. I tweaked it a little bit because some things weren't working.

### Here's how you tell NGINX to serve static assets:

```nginx
location /_next/static {
    alias /home/ubuntu/PROJECT_FOLDER/.next/static;
    add_header Cache-Control "public, max-age=3600, immutable";
}
```

### Reverse Proxy

I love and hate fancy developer language. Here's a good explanation from wikipedia:

> In computer networks, a reverse proxy is the application that sits in front of back-end applications and forwards client (e.g. browser) requests to those applications. Reverse proxies help increase scalability, performance, resilience and security. The resources returned to the client appear as if they originated from the web server itself.
- https://en.wikipedia.org/wiki/Reverse_proxy

The finished, working configuration, which lives at `/etc/nginx/sites-available/infosite.com` on my Raspberry Pi:

```nginx
server {
    server_name infosite.com www.infosite.com;

    # Serve any static assets with NGINX
    location /_next/static {
        location /_next/static/development/_devPagesManifest.json {
            proxy_pass http://localhost:3000;
        }
        location /_next/static/development/_devMiddlewareManifest.json {
            proxy_pass http://localhost:3000;
        }
        alias /home/j888/next-apps/info-site-v2/.next/static;
        add_header Cache-Control "public, max-age=3600, immutable";
    }

    location / {
	      # reverse proxy for next server
        proxy_pass http://localhost:3000; #Don't forget to update your port number
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;	

	     add_header Cache-Control "public, max-age=3600";
    }

    listen 80;
    listen [::]:80;
}
```

THANK YOU to this guy for pointing out that some files in the static directory are not really files, they're things that Next builds at runtime: https://stackoverflow.com/a/71754572

### I was getting 403 errors for anything in `./next/static`



```sh
# I'm not sure if this was needed. I did it somewhere along the way and eventually things were fixed.
sudo chmod -R 755 /home/j888/next-apps/info-site-v2/.next/
sudo chmod -R 755 /home/j888/next-apps/info-site-v2/public/
```

I think this is actually the thing that fixed my 403s. In `/etc/nginx/nginx.conf` modify this line to use your user (don't use root):

```nginx
user <YOUR_USER>;
```


### Enabling the site

Create Sym-Link, Then Restart NGINX

```sh
sudo ln -s /etc/nginx/sites-available/localinfosite-dev.com /etc/nginx/sites-enabled/

# make sure there are no syntax errors in any of the files
# If nothing is wrong, you should see: 
#     nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
#     nginx: configuration file /etc/nginx/nginx.conf test is successful
sudo nginx -t

# restart nginx
sudo systemctl restart nginx
```


## And, of course, you've gotta make sure you modify /etc/hosts to use the fake domain name you told NGINX about:

```sh
sudo vi /etc/hosts
```

Then add this line at the end:
```
<YOUR_RPi_Ip_Address> <FAKE_HOST_NAME>.com www.<FAKE_HOST_NAME>.com

# e.g. for me this line ended up being:
# 192.168.4.205 infosite.com www. infosite.com
```
