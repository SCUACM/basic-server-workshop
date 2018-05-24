From Signup to SSL
==
## Start up AWS Instance
Once logged into AWS with the provided credentials, create an EC 2 instance
- Under services in the top left, select EC2
- Select "Launch Instance" from the top left
- Pick the "Ubuntu Server 16.04 LTS (HVM), SSD Volume Type" machine image
- Leave t2.micro selected and click "Review and Launch"
- Across from Security Groups select "Edit security groups"
- Set "Security group name" field to "\<Firstname>-\<Lastname>-Wiz"
- Click "Review and Launch"
- Click "Launch"
- When prompted select "Create a new key pair" in the first selection box
- Name the keypair "\<FirstName>-\<LastName>-Keys"
- Click "Download Key Pair"
- Save the file
- Once launched click "View Instances"
- You will see the list of all instances, your new instance is denoted by your key pair which should be named as seen above. The instance may take time to spin up.

## Connecting to your instance
- Select your instance from the list and locate the "IPv4 Public IP" field, make note of this number
- Open a terminal and navigate to your downloaded key (It should be named "\<FirstName>-\<LastName>-Keys.pem")
- Move these to your local `.ssh` folder
```
mv -v <KeyPair.pem> ~/.ssh/
```
- Navigate to your `~/.ssh` folder
- Update the permissions on the keys to make sure only your account may use them. `600` means you have read-write and all other users have nothing.
```
chmod 600 <KeyPair.pem>
```
- Connect to your new machine using ssh
```
ssh -i ~/.ssh/<KeyPair.pem> ubuntu@<your machine IPv4>
```
- When prompted to add the server ECDSA fingerprint type "yes"
- You should now be connected to your new machine

## Disable Root SSH
- Become the root user (this is dangerous, everything you type is now the word of God)
```
sudo -i
```
- edit `/etc/ssh/sshd_config` with `nano`
```
nano /etc/ssh/sshd_config
```
- Change `PermitRootLogin permit-password` to `PermitRootLogin no`. This disables all direct connection to the root account.
- Save the file by pressing `Control-X`, then `y`, then hitting `Enter`
- Restart `ssh`
```
service ssh restart
```
- Logout of root and the server by using `exit` twice

## Setup NGINX
- Connect to the server again
```
ssh -i ~/.ssh/<KeyPair.pem> ubuntu@<your machine IPv4>
```
- Update package listings from apt
```
sudo apt update
```
- Install all needed upgrades
```
sudo apt upgrade
```
- Install NGINX from apt
```
sudo apt install nginx
```
- Reopen the AWS control panel and select your servers "Security Groups" entry. It will be named "\<FirstName>-\<LastName>-wiz"
- Select the "Inbound" tab
- Click "Edit" then "Add Rule"
- Enter port 80
- Click "Add Rule" again and enter 443
- Hit "Save"
- Open `http://<Your machine IPv4>/` you should see welcome to NGINX!
- Edit `/var/www/html/index.nginx-debian.html`
```
sudo nano /var/www/html/index.nginx-debian.html
```
- Modify the HTML in some way. I replaced `<h1>Welcome to nginx!</h1>` with `<h1>Welcome to Donovan's nginx!</h1>`
- Reload `http://<Your machine IPv4>/` and see your changes

## Setup and test MySql
- Install MySql
```
sudo apt install mysql-server
```
- Enter a MySql root password, I use "root"
- Connect to your new MySql server, when prompted enter the root password you selected
```
mysql -u root -p
```
- Create a new database
```
CREATE DATABASE test_db;
```
- Switch to it
```
USE test_db;
```
- Create a new table in it
```
CREATE TABLE Friends (FriendID int, FirstName varchar(255), LastName varchar(255), Age int);
```
- Add an entry
```
INSERT INTO Friends (FriendID, FirstName, LastName, Age) VALUES (1, "James", "Simon", 25);
```
- Select it
```
SELECT * FROM Friends;
```
- Exit MySql with `Control-D`
- Wow! That was some work! Lets install something to make it easier.

## Install and setup phpMyAdmin
### Install
- Get latest version of phpMyAdmin from their website or download the version we will use with
```
wget https://files.phpmyadmin.net/phpMyAdmin/4.8.0.1/phpMyAdmin-4.8.0.1-all-languages.zip
```
- Install Unzip, Php server and Php utils
```
sudo apt install unzip php7.0-fpm php7.0-mbstring php7.0-mysqli
```
- Unzip phpMyAdmin
```
unzip phpMyAdmin-4.8.0.1-all-languages.zip
```
- Relocate it to a more fitting location, I like `srv`
```
sudo mv -v phpMyAdmin-4.8.0.1-all-languages /srv/phpMyAdmin
```
- Let the `www-data` user own the phpMyAdmin
```
sudo chown -R www-data:www-data /srv/phpMyAdmin
```
### Setup
- Create a new site for nginx to serve
```
sudo nano /etc/nginx/sites-available/phpMyAdmin
```
- Setup this site
```
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /srv/phpMyAdmin;
    index index.php;
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.0-fpm.sock;
    }
}
```
- enable the site by creating a symbolic link
```
sudo ln -s /etc/nginx/sites-available/phpMyAdmin /etc/nginx/sites-enabled/phpMyAdmin
```
- Remove default site
```
sudo rm /etc/nginx/sites-enabled/default
```
- Test the new configuration (so we don't end up with a broken NGINX)
```
sudo nginx -t
```
- Reboot nginx
```
sudo service nginx restart
```
- Check the service status
```
sudo service nginx status
```
- Access and login to our new web interface at `http://<Your machine IPv4>/`

## Linking DNS
- Login to Cloudflare with the credentials provided
- Under DNS add an A record with your own name in the name field and your server IP address in the value field
- Deselect the cloud icon so the arrow points around the cloud not through it
- Add this record
- Visit `http://<Your name>.scuacm.tk/` and see that it links to your phpMyAdmin page

## Acquire SSL Certs
- Add Certbot as a trusted package supplier
```
sudo add-apt-repository ppa:certbot/certbot
```
- Update package listings
```
sudo apt update
```
- Install the certbot client
```
sudo apt install python-certbot-nginx
```
- Validate the domain
```
sudo certbot --nginx
```
- Enter acm-keys@dallen.io for the email
- When prompted, enter \<Your name>.scuacm.tk as the domain you would like to validate
- When prompted to add redirects select `1`, no redirect

## Configure NGINX to serve the certs
- Edit the phpMyAdmin site config
```
sudo nano /etc/nginx/sites-available/phpMyAdmin
```
- Modify it to use SSL
```
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name <Your name>.scuacm.tk;

    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;

    ssl_certificate /etc/letsencrypt/live/<Your name>.scuacm.tk/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<Your name>.scuacm.tk/privkey.pem;

    root /srv/phpMyAdmin;
    index index.php;
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.0-fpm.sock;
    }
}
```
- Test and restart nginx
```
sudo nginx -t
sudo service nginx restart
```
- Check `https://<Your name>.scuacm.tk/`
- Marvel at the security!

## Setup a redirect to the new SSL domain
- Edit the phpMyAdmin nginx config again
```
sudo nano /etc/nginx/sites-available/phpMyAdmin
```
- Add a new server block to handle non ssl traffic and upgrade it
```
server {
    listen 80;
    listen [::]:80;
    server_name <Your name>.scuacm.tk;

    return 301 https://$host$request_uri;
}
```
- Test and restart nginx
```
sudo nginx -t
sudo service nginx restart
```
Fin.
