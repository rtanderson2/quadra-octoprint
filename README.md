# Manually Install Octoprint On Linux – Complete Guide!

I wrote these instructions specifically for use on the Inovato Quadra. But the reality is, any hardware running a flavor of Debian should work just fine.

## Assumptions
- You haven't done anything but plugin your Quadra. 
- You know the IP address of the Quadra (I suggest setting up a reserved IP on your router)
- This guide covers setting up a USB camera but you can skip that section if you don’t plan on using one

All of these steps can be completed via the command line via SSH. You can even setup wifi via the command line, but you'll need to use your Google-fu to learn how to do that. 

## Step 1 - The User

### Create the pi user

If you don't want to use the username 'pi' here, that's fine. You'll just have to change it everywhere throughout the rest of this document.

`sudo useradd -m pi -s /bin/bash`

### Give the pi user sudo access

`usermod -aG sudo pi`

### Change to the pi user and go to the home folder

`sudo su pi`

`cd ~`

## Step 2 - Prepping For Install

### Run Updates

You should basically always do this before you start installing stuff on Linux. 

`sudo apt update`

`sudo apt upgrade`

## Step 3 - Let's Get Installing Stuff

### Install Packages

`sudo apt install python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev build-essential`

### Setup Octoprint Folder

`cd ~`

`mkdir OctoPrint && cd OctoPrint`

### Setup Your Virtual Environment

`python3 -m venv venv`

`source venv/bin/activate`

`pip install pip --upgrade`

### Install Octoprint

`pip install octoprint`

### Add pi User to Serial Ports

`sudo usermod -a -G tty pi`

`sudo usermod -a -G dialout pi`

**Note: You can test the Octproint install now by starting the service “~/OctoPrint/venv/bin/octoprint serve” and connecting to HTTP://<Your IP>:5000. You should get a setup page.**

### Set Octoprint to Automatically Start

`wget https://github.com/OctoPrint/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service`

`ExecStart=/home/pi/OctoPrint/venv/bin/octoprint`

`sudo systemctl enable octoprint.service`

**You can get the status with**

`sudo service octoprint {start|stop|restart|status }`

### Allow system rebooting, shutting down, and restarting OctoPrint from OctoPrint UI

`sudo nano /etc/sudoers`

Add the following to the end of the file

```
pi ALL=(ALL) NOPASSWD: /usr/bin/systemctl poweroff, /usr/bin/systemctl reboot, /usr/bin/systemctl restart octoprint
```

## Step 4 - Make It Easier To Access OctoPrint
       
### HAProxy Install/Setup

HAProxy is used to serve the front end of the site on port 80 and direct the traffic to the backend ports as needed.

### Install HAProxy

`sudo apt install haproxy`

### Update HAProxy Config

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

## Step 5 - The Webcam

### Check where your webcam is mounted

Chances are, your webcam mounts at /dev/video1. But we can find out easily enough. 

`sudo apt-get install v4l-utils`

`v4l2-ctl --list-devices`

You should see your webcam listed and it'll tell you where it's mounted. If it's anything other than /dev/video1 take note. 

### Build ustreamer

`cd ~`

`sudo apt install build-essential libevent-dev libjpeg-dev libbsd-dev`

`git clone --depth=1 https://github.com/pikvm/ustreamer`

`cd ustreamer`

`make`

### Set ustreamer to Autostart

`mkdir /home/pi/scripts`

`nano /home/pi/scripts/webcamDaemon`

Copy code below into webcamDaemon file just created. If your camera is mounted somewhere other than /dev/video1 you'll need to update this script accordingly. You can also change the resolution here. By default the FPS is set to 15. There's a bunch of other options for ustreamer you can checkout, too.

```  
#!/bin/bash

/home/pi/ustreamer/ustreamer --device=/dev/video1 --host=0.0.0.0 --port=8080 --format=MJPEG --resolution=1280x720 > /dev/null 2>&1

```  

### Setup Permissions on webcam File

`chmod +x /home/pi/scripts/webcamDaemon`

### Create webcamd Service

`sudo nano /etc/systemd/system/webcamd.service`

### Copy code below into webcamd service file just created

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
  
### Enable The webcamd Service

`sudo systemctl daemon-reload`
  
`sudo systemctl enable webcamd`
       
## Step 6 - Fix Your Timezone
 
### Get the timezone list if you don't know yours       

`sudo timedatectl list-timezones`

### Update to the correct timezone
       
`sudo timedatectl set-timezone Africa/Cairo`       
       
## Step 7 - Like Any Good Install, Reboot When You're Done

### Reboot
  
`sudo reboot`
  
## System Controls and Webcam URLs

**Restart OctoPrint:** sudo systemctl restart octoprint

**Restart system:** sudo systemctl reboot

**Shutdown system:** sudo systemctl poweroff

**Stream URL:** /webcam/?action=stream

**Snapshot URL:** http://127.0.0.1:8080/?action=snapshot
  
**Path to FFMPEG:** /usr/bin/ffmpeg
