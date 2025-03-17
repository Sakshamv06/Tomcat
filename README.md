# Setting Up Multiple Tomcat Instances and Deploying Web Applications

[**Step 1: Creating Web Applications**](#step-1-creating-web-applications)  
[1.1 Writing a Script to Generate Web Applications](#1.1-writing-a-script-to-generate-web-applications)  
[1.2 Generating WAR Files for Deployment](#1.2-generating-war-files-for-deployment)  

[**Step 2: Setting Up Tomcat Instances**](#step-2-setting-up-tomcat-instances)  
[2.1 Installing and Configuring Tomcat](#2.1-installing-and-configuring-tomcat)  
[2.2 Modifying server.xml for Each Instance](#2.2-modifying-serverxml-for-each-instance)  
[2.3 Configuring setenv.sh for Each Instance](#2.3-configuring-setenvsh-for-each-instance)  

[**Step 3: Automating Tomcat Startup**](#step-3-automating-tomcat-startup)  

[**Step 4: Deploying WAR Files to Tomcat**](#step-4-deploying-war-files-to-tomcat)  

[**Step 5: Configuring HAProxy for Load Balancing**](#step-5-configuring-haproxy-for-load-balancing)  

## **Step 1: Creating Web Applications** 
### **1.1 Writing a Script to Generate Web Applications**

We created a script (`app.sh`) to generate 8 different web applications with basic HTML, CSS, and JavaScript files.

```bash
#!/bin/bash

# Create 8 different web apps
for i in {1..8}
do
  # Create directories for each app
  mkdir -p "app$i/WEB-INF"
  mkdir -p "app$i/css"
  mkdir -p "app$i/js"

  # Create index.html file for the app
  cat <<EOL > "app$i/index.html"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Web App $i</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <h1>Welcome to My Web Application $i!</h1>
    <script src="js/script.js"></script>
</body>
</html>
EOL

  # Create style.css file for the app
  cat <<EOL > "app$i/css/style.css"
body {
    font-family: Arial, sans-serif;
}

h1 {
    color: blue;
}
EOL

  # Create script.js file for the app
  cat <<EOL > "app$i/js/script.js"
console.log("JavaScript is working for app $i!");
EOL

  # Create web.xml file inside WEB-INF directory
  cat <<EOL > "app$i/WEB-INF/web.xml"
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                             http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
    <display-name>My Simple Web App $i</display-name>
</web-app>
EOL

done

echo "All 8 web applications created successfully!"
```

### **1.2 Generating WAR Files for Deployment** 

After creating the applications, we package them into WAR files using the following script (`war.sh`):

```bash
#!/bin/bash

# Loop over each app directory to create WAR files
for i in {1..8}
do
  if [ -d "app$i" ]; then
    echo "Creating WAR file for app$i..."
    jar -cvf "app$i.war" -C "app$i/" .
  else
    echo "Directory app$i does not exist, skipping..."
  fi
done

echo "All WAR files created successfully!"
```

## **Step 2: Setting Up Tomcat Instances** 

### **2.1 Installing and Configuring Tomcat** 

We downloaded and extracted the Apache Tomcat 11.0.2 zip file, then copied it into separate directories for each Tomcat instance.

```bash
sudo cp -r /opt/tomcat/apache-tomcat-11.0.2/* /opt/tomcat/tc1
sudo cp -r /opt/tomcat/apache-tomcat-11.0.2/* /opt/tomcat/tc2
sudo cp -r /opt/tomcat/apache-tomcat-11.0.2/* /opt/tomcat/tc3
sudo cp -r /opt/tomcat/apache-tomcat-11.0.2/* /opt/tomcat/tc4
```

### **2.2 Modifying server.xml for Each Instance** 
We modified `server.xml` in each instance to set unique ports for each Tomcat instance:

| Tomcat Instance | Server Port | Connector Port | AJP Port |
|-----------------|-------------|----------------|----------|
| tc1             | 8005        | 8080           | 8009     |
| tc2             | 8006        | 8081           | 8010     |
| tc3             | 8007        | 8082           | 8011     |
| tc4             | 8008        | 8083           | 8012     |

Example diff between two `server.xml` files:

```bash
diff /opt/tomcat/tc3/conf/server.xml /opt/tomcat/tc2/conf/server.xml
```

### **2.3 Configuring setenv.sh for Each Instance**

We created and modified `setenv.sh` to define `CATALINA_HOME` for each instance:

```bash
export CATALINA_HOME=/opt/tomcat/tcX
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

Example diff between two `setenv.sh` files:

```bash
diff /opt/tomcat/tc3/bin/setenv.sh /opt/tomcat/tc2/bin/setenv.sh
```

## **Step 3: Automating Tomcat Startup** 

We created a script (`start_all_instances.sh`) to start all Tomcat instances:

```bash
#!/bin/bash

/opt/tomcat/tc1/bin/startup.sh &
echo "Tomcat Instance 1 started on port 8080"

/opt/tomcat/tc2/bin/startup.sh &
echo "Tomcat Instance 2 started on port 8081"

/opt/tomcat/tc3/bin/startup.sh &
echo "Tomcat Instance 3 started on port 8082"

/opt/tomcat/tc4/bin/startup.sh &
echo "Tomcat Instance 4 started on port 8083"

wait
echo "All Tomcat instances started."
```

Run the script:

```bash
chmod +x /opt/start_all_instances.sh
/opt/start_all_instances.sh
```

## **Step 4: Deploying WAR Files to Tomcat** 

We deployed the WAR files by moving them to the `webapps` directory of each Tomcat instance:

```bash
cd /home/test/Desktop/folder_with_war_files
cp app1.war app2.war /opt/tomcat/tc1/webapps/
cp app3.war app4.war /opt/tomcat/tc2/webapps/
cp app5.war app6.war /opt/tomcat/tc3/webapps/
cp app7.war app8.war /opt/tomcat/tc4/webapps/
```

## **Step 5: Configuring HAProxy for Load Balancing** 
We used HAProxy to distribute traffic among the Tomcat instances. The HAProxy configuration file is located at `/etc/haproxy/haproxy.cfg`.

```bash
cd /etc/haproxy/
vim haproxy.cfg
```

Changes in the `haproxy.cfg` file:

```txt
# Global settings
global
  log /dev/log local0
  log /dev/log local1 notice
  chroot /var/lib/haproxy
  stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
  stats timeout 30s
  user haproxy
  group haproxy
  daemon

  # SSL/TLS settings
  ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305
  ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
  ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

# Default settings
defaults
  log global
  mode http
  option httplog
  option dontlognull
  timeout connect 5000
  timeout client 50000
  timeout server 50000

# Frontend configuration
frontend http_front
  bind *:80
  option httplog
  option forwardfor
  acl app1_request path_beg /app1
  acl app2_request path_beg /app2
  acl app3_request path_beg /app3
  acl app4_request path_beg /app4
  acl app5_request path_beg /app5
  acl app6_request path_beg /app6
  acl app7_request path_beg /app7
  acl app8_request path_beg /app8
  use_backend tc1_backend if app1_request
  use_backend tc1_backend if app2_request
  use_backend tc2_backend if app3_request
  use_backend tc2_backend if app4_request
  use_backend tc3_backend if app5_request
  use_backend tc3_backend if app6_request
  use_backend tc4_backend if app7_request
  use_backend tc4_backend if app8_request

# Backend configurations for each Tomcat instance
backend tc1_backend
  server tc1 *:8080 check

backend tc2_backend
  server tc2 *:8081 check

backend tc3_backend
  server tc3 *:8082 check

backend tc4_backend
  server tc4 *:8083 check

# HAProxy Stats Configuration
listen stats
  bind *:8088
  stats enable
  stats uri /haproxy_stats
  stats auth admin:password
```

Restart HAProxy:

```bash
sudo systemctl restart haproxy
systemctl status haproxy.service
```

To monitor logs:

```bash
journalctl -xeu haproxy.service
```

To access the apps, you can try:

```txt
localhost/app1/
localhost/app2/
```
