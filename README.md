# Install Octoprint on the Inovato Quadra – Complete Guide!

## Assumptions
- You haven't done anything but plugin your Quadra and turn it on
- You know the IP address of the Quadra
- This guide covers setting up a USB camera but you can skip that section if you don’t plan on using one

All of these steps can be completed via the command line via SSH. You can even setup wifi via the command line, but you'll need to use your Google-fu to learn how to do that.

## Create the pi user

If you don't want to use the username 'pi' here, that's fine. You'll just have to change it everywhere throughout the rest of this document.

`sudo useradd -m pi -s /bin/bash`

## Give the pi user sudo access

`usermod -aG sudo pi`

## Run Updates

`sudo apt update`

`sudo apt upgrade`

## Install Packages

`sudo apt install python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev build-essential`

## Setup Octoprint Folder

`cd ~`

`mkdir OctoPrint && cd OctoPrint`

## Setup Virtual Environment

`python3 -m venv venv`

`source venv/bin/activate`

## Install Octoprint

`pip install pip --upgrade`

`pip install octoprint`

## Add pi User to Serial Ports

`sudo usermod -a -G tty pi`

`sudo usermod -a -G dialout pi`

Note: You can test the Octproint install now by starting the service “~/OctoPrint/venv/bin/octoprint serve” and connecting to HTTP://<Your IP>:5000. You should get a setup page.

## Set Octoprint to Automatically Start

`wget https://github.com/OctoPrint/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service`

`ExecStart=/home/pi/OctoPrint/venv/bin/octoprint`

`sudo systemctl enable octoprint.service`

You can get the status with

`sudo service octoprint {start|stop|restart }`

## HAProxy Install/Setup

HAProxy is used to serve the front end of the site on port 80 and direct the traffic to the backend ports as needed.

## Install HAProxy

`sudo apt install haproxy`

## Update HAProxy Config

`sudo nano /etc/haproxy/haproxy.cfg`

Add this to the bottom of the config

```  
frontend public
       bind *:80
       use_backend webcam if { path_beg /webcam/ }
       default_backend octoprint

backend octoprint
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        http-request replace-path /webcam/(.*)   /\1
        server webcam1  127.0.0.1:8080
```

## Install/Setup Webcam Support

### Build ustreamer

`cd ~`

`sudo apt install build-essential libevent-dev libjpeg-dev libbsd-dev`

`git clone --depth=1 https://github.com/pikvm/ustreamer`

`cd ustreamer`

`make`

## Set ustreamer to Autostart

`mkdir /home/pi/scripts`

`nano /home/pi/scripts/webcamDaemon`

## Copy code below into webcamDaemon file just created

```  
#!/bin/bash

/home/pi/ustreamer/ustreamer --device=/dev/video1 --host=0.0.0.0 --port=8080 --format=MJPEG --resolution=1280x720 > /dev/null 2>&1

```  

## Setup Permissions on webcam File

`chmod +x /home/pi/scripts/webcamDaemon`

## Create webcamd Service

`sudo nano /etc/systemd/system/webcamd.service`

## Copy code below into webcamd service file just created

```  
[Unit]
Description=uStreamer webcam streamer for OctoPrint
After=network-online.target OctoPrint.service
Wants=network-online.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/scripts/webcamDaemon

[Install]
WantedBy=multi-user.target
```
  
## Enable the webcamd Service

`sudo systemctl daemon-reload`
  
`sudo systemctl enable webcamd`
  
## Reboot
  
`sudo reboot`
  
## Webcam URLs

**Stream URL:** /webcam/?action=stream

**Snapshot URL:** http://127.0.0.1:8080/?action=snapshot
  
**Path to FFMPEG:** /usr/bin/ffmpeg
