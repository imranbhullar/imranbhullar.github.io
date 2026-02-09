---
layout: post
title: "Playing with Apache Tomcat"
date: 2011-09-22 10:00:00 +0500
description: "A beginner's guide to setting up and configuring Apache Tomcat, exploring the basics of Java-based web servers."
categories: [Java, Web Servers]
tags: [Apache Tomcat, Java, Web Development, Server Configuration]
---
![nix](/assets/img/Linux_unix_bsd.png)

Apache Tomcat is an open-source web server and servlet container developed by the Apache Software Foundation. It implements several Java EE specifications including Java Servlet, JavaServer Pages (JSP), Java EL, and WebSocket, and provides a "pure Java" HTTP web server environment in which Java code can run.

### Setting Up Tomcat

To get started with Tomcat, you first need to have the Java Development Kit (JDK) installed on your system. Once Java is ready, you can download the Tomcat binary distributions from the official Apache website.

1. **Download:** Get the latest stable version (e.g., Tomcat 7 or 8 depending on your requirements).
2. **Extraction:** Extract the zip or tar.gz file to a directory of your choice.
3. **Environment Variables:** Set the `CATALINA_HOME` environment variable to point to your Tomcat installation directory.

* Download JDK 7:

```bash
[root@centos garbage]# wget http://download.oracle.com/otn-pub/java/jdk/7/jdk-7-linux-i586.rpm

#Install JDK 7:

[root@centos garbage]# rpm -ivh jdk-7-linux-i586.rpm

#Verify the installation:

[root@centos garbage]# java
[root@centos garbage]# java -version
java version "1.7.0"
Java(TM) SE Runtime Environment (build 1.7.0-b147)
Java HotSpot(TM) Client VM (build 21.0-b17, mixed mode, sharing)
```
* Downloading & Installing Tomcat

```bash
#The latest stable release at the time of writing is Tomcat 7.0.21.
#Download Tomcat:

[root@centos garbage]# wget http://www.apache.org/dist/tomcat/tomcat-7/v7.0.21/bin/apache-tomcat-7.0.21.tar.gz

#Extract Tomcat:
[root@centos garbage]# tar vxzf apache-tomcat-7.0.21.tar.gz
[root@centos garbage]# cd apache-tomcat-7.0.21
```


### Basic Commands

You can control the Tomcat server using the scripts located in the `/bin` directory:

* **Start Server:** * Windows: `startup.bat`
    * Linux/Mac: `./startup.sh`
* **Stop Server:** * Windows: `shutdown.bat`
    * Linux/Mac: `./shutdown.sh`

Once started, you can verify the installation by navigating to `http://localhost:8080` in your web browser. You should see the default Tomcat welcome page.

### Deploying a Simple Application

To deploy a web application, you usually package it as a `.war` file and place it in the `webapps` directory. Tomcat automatically detects the file, extracts it, and deploys the application.

Alternatively, you can place your static HTML files or JSP files into a new folder within `webapps` to see them live instantly.

### Configuration

The primary configuration file for Tomcat is `server.xml`, located in the `/conf` directory. Here you can change the default port (8080) if it conflicts with other services:

```xml
<Connector port="8080" protocol="HTTP/1.1" 
           connectionTimeout="20000" 
           redirectPort="8443" />
```
You should see the Tomcat Administration page at `localhost:port`.

To add or remove users, navigate to the "conf" directory within the Apache Tomcat directory and edit the necessary user configurations by adding or modifying the following line as needed:

`<user name="desired_Username" password="desired_passwd" roles="admin,manager,tomcat"/>`

Apache Tomcat is an industry giant with vast complexities, but you don't need to master it all at once. We’ve only scratched the surface by touching on the basic configuration parameters required to kickstart the journey for someone with zero prior knowledge. This foundation is more than enough to get your first application up and running.