# Automatization Challange (Container-based Nginx Web Server with valid SSL Certificate)

## Part 1 - Create a CloudFormation Template

- Create a Cloud Formation template in order to create Docker machine on Amazon Linux 2 AMI with security group allowing SSH, HTTP adn HTTPS ports and save it as `webserver-cfn-template.yml´.

```bash
aws cloudformation create-stack --stack-name test --template-body file://webserver-cfn-template.yml
```

- Create a stack on AWS CloudFormation using recently created template.

- After creation completed, create a A-Record on akmustafa.net hosted zone using AWS Route 53 service console.
A-Record should be 'automation-challange.akmustafa.net'.

-  After that create a secret on our https://github.com/Musti4096/nginx-ssl.git GitHub account using server public ip address and
ssh pem key, in order to connect our server using SSH connection. This step is critical for CI/CD pipeline on GitHub Actions.

## Part 2 - Create a NGINX Docker container files

- Create a 'docker-compose.yml' file for nginx server.

```yml
version: '3.4'

services: 
  web:
    image: nginx:1.19.6-alpine
    restart: always
    volumes:
      - ./public_html:/public_html #for public html file
      - ./conf.d:/etc/nginx/conf.d/ #nginx server conf
      - ./dhparam:/etc/nginx/dhparam # store our pem key
      - ./certbot/conf/:/etc/nginx/ssl/ # store our valid ssl certificate
      - ./certbot/data:/usr/share/nginx/html/letsencrypt
    ports:
      - 80:80
      - 443:443
  
  certbot:
     image: certbot/certbot:latest
     command: certonly --webroot --webroot-path=/usr/share/nginx/html/letsencrypt --email mustafaak4096@gmail.com --agree-tos --no-eff-email -d automation-challange.akmustafa.net
     volumes:
       - ./certbot/conf/:/etc/letsencrypt
       - ./certbot/logs/:/var/log/letsencrypt
       - ./certbot/data:/usr/share/nginx/html/letsencrypt
```
- Create a simple web site, including one line 'Hello CGI' text, and save it as an index.html
under nginx-ssl/public_html directory.

```html
<html>
<body>
    <h1>Hello CGI!</h1>
</body>
</html>
```


- Then go to /nginx-ssl/dhparam directory. Create a pem key using following command.

```bash
sudo openssl dhparam -out /home/ec2-user/nginx-ssl/dhparam/dhparam-2048.pem 2048
```

- Then create a configuration file under conf.d directory, write following configuration inside of it and save it as a default.conf. 

```text
server {
    listen 80;
	server_name automation-challange.akmustafa.net;
    root /public_html/;

    location ~ /.well-known/acme-challenge{
        allow all;
        root /usr/share/nginx/html/letsencrypt;
    }
}
```

- Go to nginx-ssl directory and run docker-compose.yml file with following command.

```bash
docker-compose up -d
```

- After that command, our nginx server and certbot container runs. Also our SSL certificate created and
pass the challange. Then we should stop our containers and update our default.conf file as follows 
in order to redirect HTTP traffic to HTTPS.

```text
server {
    listen 80;
	server_name automation-challange.akmustafa.net;
    root /public_html/;

    location ~ /.well-known/acme-challenge{
        allow all;
        root /usr/share/nginx/html/letsencrypt;
    }

        location / {
        return 301 https://automation-challange.akmustafa.net$request_uri;
    }
}

server {
     listen 443 ssl http2;
     server_name automation-challange.akmustafa.net;
     root /public_html/;

     ssl on;
     server_tokens off;
     ssl_certificate /etc/nginx/ssl/live/automation-challange.akmustafa.net/fullchain.pem;
     ssl_certificate_key /etc/nginx/ssl/live/automation-challange.akmustafa.net/privkey.pem;
     ssl_dhparam /etc/nginx/dhparam/dhparam-2048.pem;
     
     ssl_buffer_size 8k;
     ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
     ssl_prefer_server_ciphers on;
     ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

    location / {
        index index.html;
    }

}
```

- After updating default.conf file, we should run our container again with following command.

```bash
docker-compose up -d
```

- Now we can reach our index.html file under automation-challange.akmustafa.net address with valid TLS certificate.


## Part 3 - Automate deploying steps and scanning docker container using GitHub Actions

- The whole previous steps are manually executed. Our code is now ready on our GitHub repository. We are ready to automize
that flows.
 
- We can create a CI/CD pipeline using GitHub Actions. First, we should click Actions button on our nginx-ssl repository,
then choose a related pipeline options. Then .github/workflows directory automatically created and we can update our
pipeline yml file as follows. In that workflow, first our code pull related container images from DockerHub, Anchor Container
Scan tool works for vulnerability scanning. 

```yml
name: WebServer Deployment Pipeline

on:
  push:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: pull the container image
      run: docker pull nginx:1.19.6-alpine
    - uses: anchore/scan-action@v2
      with:
        image: "nginx:1.19.6-alpine"
        fail-build: true
        severity-cutoff: critical
    
    - uses: actions/checkout@v2
    - name: pull certbot container image
      run: docker pull certbot/certbot
    - uses: anchore/scan-action@v2
      with:
        image: "certbot/certbot"
        fail-build: true
        severity-cutoff: critical

    - name: Deploy Webserver
      uses: appleboy/ssh-action@v0.1.2
      with:
        host: ${{secrets.SSH_HOST}}
        username: ec2-user
        key: ${{ secrets.PEM_SECRET }}
        port: 22
        script: |
          sudo git clone https://github.com/Musti4096/nginx-ssl.git
          sudo cd nginx-ssl/dhparam
          sudo openssl dhparam -out /home/ec2-user/nginx-ssl/dhparam/dhparam-2048.pem 2048
          cd nginx-ssl/
          docker-compose up -d
          sleep 6
          docker-compose down
          sleep 6
          pwd
          cd conf.d/
          sleep 2
          sudo cp second-step.txt default.conf
          cd ..
          sleep 6
          docker-compose up -d
```

- When create our server with CloudFormation template, we should get our server's public ip address and update that on GitHub
secrets and on our A-Record.

- Then we push our code on master branch. In couple of minutes later, we can reach our webserver under
automation-challange.akmustafa.net address with valid TLS certificate.

- After successfully create our server, we should update our CI/CD pipeline yaml file as in follows, in order to change just our web site code. 

```yml
name: WebServer Deployment Pipeline

on:
  push:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: pull the container image
      run: docker pull nginx:1.19.6-alpine
    - uses: anchore/scan-action@v2
      with:
        image: "nginx:1.19.6-alpine"
        fail-build: true
        severity-cutoff: critical
    
    - uses: actions/checkout@v2
    - name: pull certbot container image
      run: docker pull certbot/certbot
    - uses: anchore/scan-action@v2
      with:
        image: "certbot/certbot"
        fail-build: true
        severity-cutoff: critical

    - name: Deploy Webserver
      uses: appleboy/ssh-action@v0.1.2
      with:
        host: ${{secrets.SSH_HOST}}
        username: ec2-user
        key: ${{ secrets.PEM_SECRET }}
        port: 22
        script: |
          cd nginx-ssl/public_html
          sudo rm -f index.html
          sudo wget https://raw.githubusercontent.com/Musti4096/nginx-ssl/master/public_html/index.html
```