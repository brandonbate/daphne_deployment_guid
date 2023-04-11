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
Redis is automatically added a systemd service, which is nice!

### Step 4
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
server {
        listen 80;
        server_name tictactoe.bearcornfield.com;

        location / {
                proxy_pass http://localhost:8000;
                proxy_set_header Host $host;
        }
}
```
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
I confirmed nginx was running by visiting ```tictactoe.bearcornfield.com```.

### Step 5
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
