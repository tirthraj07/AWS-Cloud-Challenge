# Deploying a Web Application on EC2 Step by Step

### Step 1: Navigate to EC2 Dashboard
In the AWS Management Console, type "EC2" in the search bar and select "EC2" from the dropdown menu.  
This will take you to EC2 Dashboard

![step1](public/step1.png)

### Step 2: Click on launch instance
  
![step2](public/step2.png)

### Step 3: Configure the Instance

#### 3.1 Select Name
![step3.1](public/step3.png)

#### 3.2 Choose Amazon Machine Image 
For our purpose, we can use the Ubuntu Server  

![step3.2](public/step4.png)

#### 3.3 Choose an Instance type
Select the "t2.micro" instance type.  

This instance type is part of the free tier for eligible users, offering 750 hours per month for the first 12 months.  

![step3.3](public/step5.png)

#### 3.4 Select a Key Pair
You need a key pair to SSH into your instance.  

If you already have a key pair, select it. If not, create a new key pair and download it (.pem file). Keep this file secure as you’ll need it to access your instance.

![step3.4.1](public/step6.png)

![step3.4.2](public/step7.png)

#### 3.5 Configure Security Group
A security group acts as a virtual firewall to control inbound and outbound traffic.
Create a new security group with following rules.

![step3.5](public/step8.png)

#### 3.6 Launch Instance
Click on launch instance

![step3.6](public/step9.png)


### Step 4: Connect to EC2 instance using EC2 Instance Connect

![step4.1](public/step10.png)

![step4.2](public/step11.png)

This should connect you to the virtual machine.

You can also connect to the instance using the ssh key we generated earlier (Optional)

![step4.3](public/step12.png)

### Step 5: Update package list and install dependencies

#### 1. Setup the EC2 instance

```bash
sudo apt update
sudo apt upgrade
```

#### 2. Install NodeJS and npm
Add the Node.js PPA (Personal Package Archive) to get the latest version of Node.js:

```bash 
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
```

Then install Node.js and npm:
```
sudo apt install nodejs -y
```

---

Verify Installation of Node.js and npm

Check Node.js version
```bash
node -v
```
Check npm version
```bash
npm -v
```

If npm is not installed for some reason, you can install it separately.
```
sudo apt install npm -y
```

---

#### 3. Install Git
You need to install git to bring your files into virtual machine.
If you are going to `scp` (Secure copy protocol) for transferring your files, you can skip this step

```bash
sudo apt install git -y
```

### Step 6: Transfer the project files
In this step, you need to transfer all the project files using either git or scp or any other mechanism

For this example, I'll use git

Navigate to the directory where you want to clone your project. For example, your home directory:

```bash
cd ~
```

Clone your repository

```bash
git clone https://github.com/your-username/your-repo-name.git
```
Replace your-username and your-repo-name with your GitHub username and repository name respectively.

![step6.1](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step13.png?raw=true)

![step6.2](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step14.png?raw=true)


### Step 7: Install project dependencies

#### 1. Navigate to your Project Directory
```bash
cd your-repo-name
```

#### 2. Install Project Dependencies
Use npm to install the project dependencies defined in your package.json file:

```bash
npm install
```

![step7](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step15.png?raw=true)

You cannot directly run your application due to the following reasons:
1. We need to set up the environment variables
2. We need the application to run in the background. If we close the terminal, the server will shut down
3. Our application is running on port 3000. The Firewall does not accept connections to that port as we haven't enabled it in our security group settings

### Step 8: Setup environment variables

#### 1.  Create the Environment File
```bash
sudo vim /etc/app.env
```

#### 2. Add your variables
In Vim, add your variables in the format VARIABLE=value. For example:
```
PORT=3000
```

> Press `i` to enter insert mode in vim
> Paste the environment variables
> Press Escape to exit the mode
> Enter: `:x` to save and exit

![step8](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step16.png?raw=true)

#### 3. Restrict the file permissions for security.
```bash
sudo chmod 600 /etc/app.env
sudo chown ubuntu:ubuntu /etc/app.env
```

### Step 9: Run the server in background

We want our node.js server to run in the background, i.e.: when we close our terminal we want our server to keep running

For this purpose, we need a `Daemon Service`
A daemon is a background process or program that is designed to perform tasks without any direct user intervention

We can create a daemon service using `systemd`.

Nearly every Linux distro comes with systemd, which means forever, monit, PM2, etc are no longer necessary - your OS already handles these tasks.

Creating a `daemon service` with `systemd` involves writing a `script` for the daemon process and creating a service file for systemd to manage it.

A systemd service file is a unit configuration file that defines how systemd starts and manages a certain background service.

#### 1. Create the systemd Service File
Navigate to the systemd directory and create a new service file, `myapp.service`.

```bash
sudo vim /etc/systemd/system/myapp.service
```

#### 2. Define the service settings
Add the following content in Vim, modifying as needed for your application.


```bash
[Unit]
Description=Node.js App
After=network.target multi-user.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=/usr/bin/npm start
Restart=always
Environment=NODE_ENV=production
EnvironmentFile=/etc/app.env
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```
  

Change the `Description` as per your app name.  
Change the `WorkingDirectory` to `/home/ubuntu/your-repo-name`

There are different ways to start a Node.js application, depending on your project setup.
Change the `ExecStart` to `/usr/bin/project_start_command`
For example: `npm start` or `node server.js` or `nodemon server.js`

Change the `SyslogIdentifier` to project name for logging purposes



![step9.1](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step17.png?raw=true)

  
#### 3. Reload systemd and start your service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
```

Verify that the service is running properly.

```bash
sudo systemctl status myapp.service
```

![step9.2](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step18.png?raw=true)

---

### Step 10: Set Up a Reverse Proxy with Nginx

A __reverse proxy__ is a server that __sits between client devices and web servers__, forwarding client requests to the appropriate server. 

When a client sends a request to a website, the reverse proxy __intercepts__ this request and forwards it to the backend server (in this case, the Node.js application running on port 3000). 

The backend server processes the request and sends the response back to the reverse proxy, which then forwards it to the client.

Why a reverse proxy is commonly used and why the Node.js application was not directly run on ports 80 or 443:

1. On many operating systems, binding to ports below 1024 (such as 80 and 443) requires elevated privileges (root access)
2. A reverse proxy can distribute incoming requests across multiple instances of the Node.js application, improving load balancing and allowing the system to scale horizontally
3. Running the application behind a reverse proxy helps isolate it from direct exposure to the internet. This provides an additional layer of security.

A reverse proxy like Nginx or Apache is more efficient at handling a large number of incoming requests, static content, and SSL/TLS encryption.

In this tutorial, I'll use `Nginx` for reverse proxy
  
#### 1. Install Nginx
```bash
sudo apt install nginx
```

#### 2. Start and Enable Nginx
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

![step10.1](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step19.png?raw=true)


#### 3. Verify that Nginx is installed

Go to the VM's public IP Address. You can find that in `network` of the instance dashboard

![step10.2](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step20.png?raw=true)

![step10.3](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step21.png?raw=true)


If you see this, it means that nginx is installed correctly

#### 4. Configure Nginx as a Reverse Proxy
Create a new configuration file for your Node.js application. You can place this file in the `/etc/nginx/sites-available` directory. For example, create a file called `nodeapp`

```bash
sudo nano /etc/nginx/sites-available/nodeapp
```

Add the following configuration to the file:

```bash
server {
    listen 80;
    server_name your_domain_or_IP;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

![step10.4](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step22.png?raw=true)

#### 5. Enable the Configuration

Create a symbolic link to enable the configuration:
```bash
sudo ln -s /etc/nginx/sites-available/nodeapp /etc/nginx/sites-enabled/
```

Test the Nginx configuration for syntax errors:
```bash
sudo nginx -t
```

If there are no errors, restart Nginx to apply the changes:
```bash
sudo systemctl restart nginx
```

#### 6. Adjust Firewall Rules
Ensure that your firewall allows HTTPS traffic:
```bash 
sudo ufw allow 'Nginx Full'
```
  
   
![step10.5](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step23.png?raw=true)


#### 7. Verify the setup

Open your web browser and navigate to `http://IP`. 
You should see your Node.js application, indicating that Nginx is successfully proxying requests to your application running on port 3000.

![step10.6](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step24.png?raw=true)

---
### Step 11: Obtain SSL/TLS Certificate (Optional Step)

You need to obtain a SSL/TLS Certificate inorder to use HTTPS instead of HTTP.

Also we need to configure nginx such that, if the user visits __http__ then they should be redirect to __https__

There are two approaches:
1. __Using Let's Encrypt with a Domain Name__
2. __Creating a Self-Signed Certificate__

### 11.1 Using Let's Encrypt with a Domain Name
For this step, you require a domain name because many Certificate Authorities, including `Let's Encrypt`, prefer or require a domain name to issue SSL/TLS certificates

#### 1. Purchase a Domain Name: 
You can buy a domain name from registrars like Namecheap, GoDaddy, or Google Domains

#### 2. Point the Domain to Your Server:
Set up an A record in your domain's DNS settings to point to your server's IP address. You can do this using __Amazon Route 53__

#### 3. Complete the Configuration:

##### 1. Install Certbot and the Nginx plugin
Certbot will help you obtain and manage SSL/TLS certificates from Let's Encrypt:

```bash
sudo apt install certbot python3-certbot-nginx
```

##### 2. Obtain an SSL/TLS Certificate
Obtain an SSL/TLS Certificate

```bash 
sudo certbot --nginx -d your_domain_or_IP
```

##### 3. Configure Nginx to Redirect HTTP to HTTPS
Open the Nginx configuration file for your site:

```bash
sudo nano /etc/nginx/sites-available/nodeapp
```

Modify the configuration to include both the HTTP to HTTPS redirect and the HTTPS server block. Your updated configuration should look like this:

```bash
server {
    listen 80;
    server_name your_domain_or_IP;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your_domain_or_IP;

    ssl_certificate /etc/letsencrypt/live/your_domain_or_IP/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain_or_IP/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Make sure to replace `your_domain_or_IP` with your actual domain name or IP address

##### 4. Enable the Configuration and Restart Nginx

```bash
sudo ln -s /etc/nginx/sites-available/nodeapp /etc/nginx/sites-enabled/
```

##### 5. Test the Nginx configuration

```bash
sudo nginx -t
```

##### 6. If there are no errors, restart Nginx to apply the changes:
```bash
sudo systemctl restart nginx
```

##### 7. Adjust Firewall Rules (if necessary)
```bash
sudo ufw allow 'Nginx Full'
```

##### 8. Verify the setup
Open your web browser and navigate to `http://your_domain`. You should be automatically redirected to `https://your_domain` and see your __Node.js application served securely__.

---

### 11.2 Creating a Self-Signed Certificate

#### 1. Generate a Self-Signed Certificate:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
  
#### 2. Create a Strong Diffie-Hellman Group:
```bash
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
  
#### 3. Configure Nginx for the Self-Signed Certificate:

Create a new configuration file:

```bash
sudo nano /etc/nginx/snippets/self-signed.conf
```

Add the following configuration:

```bash
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

Create another configuration file for SSL settings:

```bash
sudo nano /etc/nginx/snippets/ssl-params.conf
```

Add the following configuration:

```nginx
ssl_protocols TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/ssl/certs/dhparam.pem;
ssl_ciphers 'ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256'; # Example cipher suite, adjust as necessary
ssl_ecdh_curve secp384r1;
ssl_session_timeout 10m;
ssl_session_cache shared:SSL:10m;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```

![step11.2.1](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step25.png?raw=true)

#### 4. Modify the Nginx Configuration File:

Edit your site configuration:

```bash
sudo nano /etc/nginx/sites-available/nodeapp
```

Update it to use the self-signed certificate:

```nginx
server {
    listen 80;
    server_name 52.87.198.144;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name 52.87.198.144;

    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

![step11.2.2](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step26.png?raw=true)

#### 5. Test and Restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

#### 6. Adjust Firewall Rules (if necessary)
Ensure that your firewall allows HTTPS traffic:

```bash
sudo ufw allow 'Nginx Full'
```

#### 7. Verification
Open your web browser and navigate to `http://your_ip_address`. You should be automatically redirected to `https://your_ip_address`, and your Node.js application should be served securely. 


![step11.2.3](https://github.com/tirthraj07/deploy-nodejs-on-ec2/blob/main/public/step27.png?raw=true)

 
> Note that with a self-signed certificate, you'll receive a browser warning about the certificate not being trusted. This is normal and expected.
> If you prefer to avoid browser warnings, consider obtaining a domain name and using Let's Encrypt to obtain a free, trusted SSL certificate.


---

There you go! You have successfully deployed your __Node JS__ project on __AWS EC2 Instance__ with __NGINX__ AND __SSL__