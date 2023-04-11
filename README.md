# Daphne Deployment Guide

I was able to deploy the tictactoe game using a Lightsail instance.
Unlike prior Lightsail instances, I had this instance use Ubuntu 22.04.
Here's how I deployed the game:

### Step 1
On the AWS console, I opened my Lightsail instance. I clicked on "Networking" tab
and created a static ip. I then associated this ip address with the ```tictactoe``` subdomain of
```bearcornfield.com```. 
I then clicked the "Connect" tab, scrolled down and clicked to "Download default key".
I placed this file in my ```WebProgramming``` folder, which contains my various projects.
I use PuTTY, so I needed to convert the ```.pem``` file I just downloaded to ```.ppk``` using PuTTYgen.
I started PuTTYgen and clicked "Load" to open the ```.pem``` file and then clicked "Save private key"
to create a corresponding ```.ppk``` file.
I then opened PuTTY and entered the address for my site ```tictactoe.bearcornfield.com```
and navigated to "Connection > SSH > Auth > Credentials" in the menu.
I then clicked "Browse" for "Private key file for authentication" and selected the ```.ppk``` file I just generated.
I clicked "session" in the menu and saved this profile. I highly recommend doing this.

### Step 2
I clicked the "Open" button on PuTTY and entered the username ```ubuntu```.
I then ran the following to update the system:
```
sudo apt update
sudo apt upgrade
sudo reboot
```
I logged back in and created GitHub SSH keys and associated them to my account:
```
ssh-keygen -t ed25519 -C "brandonbate@gmail.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```
I then added the public key (displayed in the terminal) to my GitHub account by clicking
on my Avatar and selecting "Settings > SSH and GPG keys" and clicking the "New SSH key" button.
I pasted the public key from the terminal in for the key and gave a nice title.

### Step 3
Back on PuTTY, I cloned my Github repository:
```
git clone git@github.com:brandonbate/tictactoe.git
```
I then installed redis and created a virtual environment:
```
sudo apt install redis
sudo apt install python3.10-venv
cd tictactoe
python3 -m venv virtualenv
. virtualenv/bin/activate
pip install "channels-redis==3.4.1"
pip install "daphne==4.0.0"
```
Redis is automatically added as a systemd service, which is nice!

### Step 4
I then wanted to get a quick (insecure) version of my site up and running.
I ran
```
sudo nano supertictactoe/settings.py
```
and edited the ```ALLOWED_HOSTS``` list:
```
ALLOWED_HOSTS = ['*']
```
I then ran the following:
```
sudo ./virtualenv/bin/daphne -b 0.0.0.0 -p 80 supertictactoe.asgi:application
```
I then visited my site and confirmed that my game was working.
When I first attempted this, it was a complete failure. Unbeknownst to me, the order of imports in ```asgi.py``` matters greatly.
In particular, I needed to put all imports of ```tictactoe``` files/modules AFTER calling ```django_asgi_app = get_asgi_application()```.

### Step 5
This next step is nearly identical to Step 8 of the prior Deployment Guide.
On my local machine, I opened up ```settings.py``` and replaced
```
## SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'hy=ydw9b6f57nau_#u+%hh4819!lh0my$!ep#hfci=iw#hni(c'

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

ALLOWED_HOSTS = ['*']
```
with
```
import os

if 'DJANGO_DEBUG_FALSE' in os.environ:
    DEBUG = False
    SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
    ALLOWED_HOSTS = [os.environ['SITENAME']]
else:
    DEBUG = True
    SECRET_KEY = 's#x5!*1d^7zlvbuob&=jr7dbwj%+gi+cd0cdbxo83(ls052jor'
    ALLOWED_HOSTS = []
```
Then at the local machine command line, I appended ```.gitignore``` with the following command:
```
echo .env >> .gitignore
```
I committed these changes to the repository and pushed them to GitHub.
I wen to my Lightsail console and pulled these changes.
In my project folder, I ran ```nano``` and entered the following:
```
DJANGO_DEBUG_FALSE=y
SITENAME=your_name.bearcornfield.com
DJANGO_SECRET_KEY=$(python3.7 -c"import random; print(''.join(random.SystemRandom().choices('abcdefghijklmnopqrstuvwxyz0123456789', k=50)))")
```
I saved this file as ```.env```.

### Step 6
Next, I installed nginx:
```
sudo apt install nginx
```
I then navigated to ```/etc/nginx/```. Unlike in our earlier install, the
```sites-available``` and ```sites-enabled``` directories are already present.
I deleted the ```default``` file in each folder:
```
sudo rm sites-available/default
sudo rm sites-enabled/default
```
I then ran
```
sudo nano sites-available/tictactoe.bearcornfield.com
```
and entered the following and saved:
```
upstream channels-backend {
    server localhost:8000;
}

server {
    location / {
        try_files $uri @proxy_to_app;
    }

    location @proxy_to_app {
        proxy_pass http://channels-backend;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}

```
The above configuration file was taken directly from the Channels Tutorial deployment guide.
I then created a symbolic link in ```sites-enabled```:
```
sudo ln -s sites-available/tictactoe.bearcornfield.com sites-enabled/tictactoe.bearcornfield.com
```
I then edited ```nginx.conf```:
```
sudo nano nginx.conf
```
I changed ```user www-data;``` to ```user ubuntu;```.
I then reloaded nginx:
```
sudo systemctl reload nginx
```
I then returned to the repository directory and ran the following commands:
```
set -a; source .env; set +a
./virtualenv/bin/daphne -b 0.0.0.0 -p 8000 supertictactoe.asgi:application
```
Tictactoe worked!

### Step 7

The Channels documentation recommends we use a utility called ```supervisord``` to have Daphne run persistently on the server.
I'm opting instead to use ```systemd``` (as we did earlier). In our prior deployment, we used unix sockets for nginx and gunicorn
to communicate with each other. I'm going to be lazy and just use the local port 8000. This can, of course, blow up in your face
if you allow port 8000 external access.

To configure ```systemd```, I ran
```
sudo nano /etc/systemd/system/daphne.service
```
I then entered the following and saved:
```
[Unit]
Description=Daphne
After=network.target

[Service]
Restart=on-failure
User=ubuntu
WorkingDirectory=/home/ubuntu/tictactoe
EnvironmentFile=/home/ubuntu/tictactoe/.env
ExecStart=/home/ubuntu/tictactoe/virtualenv/bin/daphne -p 8000 supertictactoe.asgi:application


[Install]
WantedBy=multi-user.target
```
I then ran the following commands to get the daphne service running:
```
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl enable daphne
sudo systemctl start daphne
```
Try rebooting the system to confirm your program still runs automatically.
