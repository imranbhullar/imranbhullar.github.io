---
layout: post
title: "Nagios on Ubuntu"
date: 2012-01-20 10:00:00 +0000
categories: [Tutorials, Linux]
tags: [nagios, ubuntu, monitoring]
---

Nagios is a powerful monitoring system that enables organizations to identify and resolve IT infrastructure problems before they affect critical business processes. In this post, I will walk through the basic steps to get Nagios up and running on an Ubuntu system.

### Prerequisites

Before starting, ensure your package database is up to date:

```bash
sudo apt-get update
```
Install Required Packages
You will need to install several dependencies, including Apache, PHP, and build libraries:

```Bash
sudo apt-get install build-essential apache2 php5-gd wget libgd2-xpm libgd2-xpm-dev libapache2-mod-php5 libssl-dev
```
Create User and Group
Create a nagios user and a group for allowing external commands to be passed through the web interface:

```Bash
sudo useradd -m -s /bin/bash nagios
sudo passwd nagios

sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios
sudo usermod -a -G nagcmd www-data
```
Download and Extract Nagios
Download the Nagios Core and Plugins source code (adjust version numbers as necessary):

```Bash
mkdir ~/downloads
cd ~/downloads
wget [http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-3.3.1.tar.gz](http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-3.3.1.tar.gz)
wget [http://prdownloads.sourceforge.net/sourceforge/nagiosplug/nagios-plugins-1.4.15.tar.gz](http://prdownloads.sourceforge.net/sourceforge/nagiosplug/nagios-plugins-1.4.15.tar.gz)

tar xzf nagios-3.3.1.tar.gz
cd nagios
```
Compile and Install
Run the configuration script and compile the binaries:

```Bash
./configure --with-command-group=nagcmd
make all
sudo make install
sudo make install-init
sudo make install-config
sudo make install-commandmode
```
Configure the Web Interface
Install the Nagios web config file in the Apache conf.d directory:

```Bash
sudo make install-webconf
```
Create a nagiosadmin account for logging into the Nagios web interface:

```Bash
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
```
Restart Apache to make the new settings take effect:

```Bash
sudo /etc/init.d/apache2 reload
```
Install Nagios Plugins
Now, compile and install the plugins:

```Bash
cd ~/downloads
tar xzf nagios-plugins-1.4.15.tar.gz
cd nagios-plugins-1.4.15

./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
sudo make install
Start Nagios
Verify the configuration and start the service:
```

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
sudo service nagios start
```
You can now access the Nagios web interface by navigating to `http://localhost/nagios/` and logging in with the nagiosadmin credentials you created earlier.