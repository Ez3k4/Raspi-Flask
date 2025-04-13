# Raspi-Flask
Instructions to create a Raspi that hosts a VM (ubuntu22.04) in Proxmox that runs Nginx with a Flask app

# Raspi with Proxmox virtualization running ubuntu 22.04 server with Nginx and Flask

## Preparing Raspi
Raspberry pi imager: https://www.raspberrypi.com/software/
- install Raspberry Pi OS Lite Bookworm 64-bit
- enable ssh 

ssh into your raspi:
```
$ sudo apt update
$ sudo apt upgrade
$ sudo apt install curl
```

give raspi a static IP address:
- using nmtui (https://pimylifeup.com/raspberry-pi-static-ip-address/#static-ip-address-using-network-manager-cli)
```
$ sudo systemctl restart NetworkManager
```

update /etc/hosts file that hostname points to IP address of my raspi
```
$ sudo nano /etc/hosts
```
set password for root usr
```
$ sudo passwd root
```

## setting up stuff for proxmox
need to add proxmox ports repo to make proxmox(initially for amd64 x86_64 systems) compatible with the raspis arm64 architecture, to do this safely need GPG
```
$ curl -L https://mirrors.apqa.cn/proxmox/debian/pveport.gpg | sudo tee /usr/share/keyrings/pveport.gpg >/dev/null
$ echo "deb [arch=arm64 signed-by=/usr/share/keyrings/pveport.gpg] https://mirrors.apqa.cn/proxmox/debian/pve bookworm port" | sudo tee  /etc/apt/sources.list.d/pveport.list
```

### install proxmox dependecies
```
$ sudo apt install ifupdown2 (handle network)
```
- create a new network interface for the virtual machines
```
$ sudo nano /etc/network/interfaces
```
```
auto eth0
  iface eth0 inet manual

auto vmbr0
iface vmbr0 inet manual
        address <IPADDRESS>
        gateway <GATEWAY>
        netmask 255.255.255.0
        bridge-ports eth0
        bridge-stp off
        bridge-fd 0
```
```
$ sudo apt install proxmox-ve postfix open-iscsi pve-edk2-firmware-aarch64
```
access proxmox over browser under 192.168.0.188:8006 
- install iso
```
$ cd /var/lib/vz/template/iso
$ sudo wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img
```
follow this in the proxmox interface to set up first empty space then load iso
https://raspberrytips.com/proxmox-on-raspberry-pi/

## install nginx
- ssh in the ubuntu2204 vm, adjust the firewall to let Nginx and OpenSSH (for raspi) pass through
```
$ sudo apt install nginx
$ sudo ufw app list 
$ sudo ufw allow 'Nginx HTTP'
$ sudo ufw allow 'OpenSSH'
$ sudo ufw status
$ sudo ufw enable
$ systemctl status nginx
```
check webserver in browser 
	- 192.168.0.103:80 (can change on reboot), should show nginx

## install flask
- install python + dependencies (pip...)
```
$ sudo apt update
$ sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
$ sudo apt install python3-venv
```
- create directory folder

```
$ sudo mkdir flaskpi
$ cd flaskpi
$ python3 -m venv flaskpi_env
$ source flaskpi_env/bin/activate
```
- install wheel to make sure packages install properly
```
$ pip install wheel
```
- install gunicorn
```
$ pip install gunicorn flask
```
create the .py file (in the folder with the virtual env)
```
$ nano webapp_raspi.py
```

```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "<h1 style='color:blue'>Hello There!</h1>"

if __name__ == "__main__":
    app.run(host='0.0.0.0')

```
- change firewall rules
```
$ sudo ufw allow 5000
```
test app => should show 'Hello There' in blue in your browser
```
$ python webapp_raspi.py
```
create a WSGI entry point to instruct gunicorn server how to interact with app
```
$ nano ~/myproject/wsgi.py
```

```
from myproject import app

if __name__ == "__main__":
    app.run()

```
```
$ gunicorn --bind 0.0.0.0:5000 webapp_raspi:app
```
- check server (192.168.103:5000) for webpage
- exit venv 
- create unit (service config) file to tell ubuntu how to manage the application
```
$ sudo nano /etc/systemd/system/flaskpi.service
```

```
[Unit]
Description=Gunicorn instance to serve flaskpi
After=network.target

[Service]
User=emil
Group=www-data
WorkingDirectory=/home/emil/flaskpi
Environment="PATH=/home/emil/flaskpi/flaskpi_env/bin"
ExecStart=/home/emil/flaskpi/flaskpi_env/bin/gunicorn --workers 3 --bind unix:flaskpi.sock -m 007 webapp_raspi:app

[Install]
WantedBy=multi-user.target
```
- gunicorn is now up and running waiting for requests, now configure nginx to pass web requests to that socket
```
$ sudo nano /etc/nginx/sites-available/flaskpi
```

```
server {
    listen 80;
    server_name niamod_domain niamod_domain; # for local development

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/emil/flaskpi/flaskpi.sock;
    }
}

```
- enable nginx server block by linkin it to the sites active
```
$ sudo ln -s /etc/nginx/sites-available/flaskpi /etc/nginx/sites-enabled
```
- remove potential conflicting files from enabled folder
- make sure everything in the path to socket is readable/ accesable by nginx
```
$ sudo chmod 755 /home/emil
$ sudo chmod 755 /home/emil/flaskpi
```
check ip for the webinterface of webapp_raspi.py

## main sources:
https://pimylifeup.com/raspberry-pi-proxmox/
https://raspberrytips.com/proxmox-on-raspberry-pi/
https://getlabsdone.com/how-to-install-ubuntu-server-22-04-on-proxmox/
https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04
https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-22-04
