# Geoserver Installation Guide

## _Deploying on Ubuntu 22.04 LTS with Tomcat 9.0.83, GeoServer 2.24.1 and PostgreSQL 14.10 with PostGIS 3.2_

GeoServer serves as an open-source server, facilitating the sharing of geospatial data. Notably designed for interoperability, it enables seamless publication of data from diverse data sources using open standards.

### Author Information
- **Author:** Daniel Pinto Salazar
- **Tested Platform:** Ubuntu 22.04 LTS
- **Software Versions:** Tomcat 9.0.83, GeoServer 2.24.1, PostgreSQL 14.10, PostGIS 3.2
- **Tested Date:** December 2023

### Software Versions

|    Software   |    Version    |
| ------------- |:-------------:|
| Ubuntu        | 22.04 LTS     |
| Java          | 17.0.9        |
| Tomcat        | 9.0.83        |
| GeoServer     | 2.24.1        |
| PostgreSQL    | 14.10         |
| PostGIS       | 3.2           |


## Installation Steps

### Step 1: Create a Droplet (Ubuntu 22.04 LTS)

First, update the packages list on Ubuntu:

```sh
sudo apt-get update
```

Set the Firewall to allow traffic from SSH, HTTP, and HTTPS:

```sh
sudo ufw allow 22
sudo ufw enable
```

### Step 2: Install Java 17 dependencies

Install OpenJDK 17:

```sh
sudo apt install openjdk-17-jdk openjdk-17-jre
```

Once installed, verify the Java version:

```sh
sudo java -version
```

### Step 3: Install Servlet Container for Java code (Tomcat)

#### Step 3.1: Create a Tomcat group and user

```sh
sudo groupadd tomcat
sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat
```

#### Step 3.2: Install Tomcat

Navigate to the temporary folder to download and extract [Tomcat 9.0.83](https://gis.stackexchange.com/questions/389555/geoserver-not-compatible-with-tomcat-10):

```sh
cd /tmp
curl -O https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.83/bin/apache-tomcat-9.0.83.tar.gz
sudo mkdir /opt/tomcat/
sudo tar xzvf apache-tomcat-9.0.83.tar.gz -C /opt/tomcat/ --strip-components=1
```

#### Step 3.3: Update Tomcat Permissions

Adjust permissions for Tomcat directories:

```sh
cd /opt/tomcat/
sudo chgrp -R tomcat /opt/tomcat/
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/
```

#### Step 3.4: Create a systemd Service File

To enable running Tomcat as a service, we'll create a systemd service file. To ensure Tomcat functions properly, it requires knowledge of the Java installation directory, often denoted as 'JAVA_HOME.' The most convenient method to find this location is by executing the following command:

```sh
sudo update-java-alternatives -l
```

To create the accurate JAVA_HOME variable, you can build it by extracting the output found in the final column the suitable JAVA_HOME for this server would be:

>JAVA_HOME 
>/usr/lib/jvm/java-1.17.0-openjdk-amd64

Using this data, we're ready to generate the systemd service file. Access the /etc/systemd/system directory and create a file named tomcat.service by executing the following command:"

```sh
sudo nano /etc/systemd/system/tomcat.service
```

Copy and insert the below content into your service file. Adjust the JAVA_HOME value, if needed, to correspond with the value you discovered on your system. Additionally, consider adjusting the memory allocation settings specified in CATALINA_OPTS. In previous Tomcat versions, remember to append the '/jre' extension to JAVA_HOME.

```sh
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.17.0-openjdk-amd64
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Upon completion, save and close the file.

Subsequently, refresh the systemd daemon to ensure recognition of our service file:

```sh
sudo systemctl daemon-reload
```

Initiate the Tomcat service using the command:

```sh
sudo systemctl start tomcat
```

Verify its startup without any errors by executing:

```sh
sudo systemctl status tomcat
```

#### Step 3.5: Adjust the Firewall and Test the Tomcat Server

Having initiated the Tomcat service, it's essential to confirm the availability of the default page.

Prior to testing, adjust the firewall settings to permit inbound requests to reach the service. Assuming you've fulfilled the prerequisites, your ufw firewall is currently active.

Tomcat typically operates on port 8080 for handling standard requests. To allow traffic through this port, execute the following command:

```sh
sudo ufw allow 8080
```

With the firewall settings updated, access the default splash page by entering your domain or IP address followed by :8080 in a web browser:

>Open in web browser 
>http://server_domain_or_IP:8080

You'll encounter the default Tomcat splash page along with other relevant information. However, certain links such as the Manager App might display access denial. We'll configure this access in the subsequent steps.

If you successfully accessed Tomcat, it's prudent to enable the service file, ensuring that Tomcat initializes automatically upon system boot:

```sh
sudo systemctl enable tomcat
```

#### Step 3.6: Configure Tomcat's Web Management Interface

To utilize Tomcat's manager web app, it's necessary to establish a login system within our Tomcat server. This involves modifying the 'tomcat-users.xml' file:

```sh
sudo nano /opt/tomcat/conf/tomcat-users.xml
```

Include a user with access to manager-gui and admin-gui (both web apps that accompany Tomcat). Insert a user definition, similar to the example below, within the tomcat-users tags. Ensure to customize the username and password for enhanced security:

```sh
<tomcat-users . . .>
    <user username="admin" password="password" roles="manager-gui,admin-gui,manager-script,manager-jmx,manager-status"/>
</tomcat-users>
```

Save and close the file after making the modifications.

By default, Tomcat's manager and host-manager applications are limited to localhost. To enable access from remote systems, adjustments need to be made to specific configuration files.

You can either permit a specific remote system or grant access from any IP. Edit the context.xml file for both manager and host-manager applications:

For the Manager app:

```sh
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```

Allow all '.*' within the IP address restriction section to enable connections from any source.

```sh
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow=".*" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```

For the Host Manager app:

```sh
sudo nano /opt/tomcat/webapps/host-manager/META-INF/context.xml
```

Allow all '.*' within the IP address restriction section to enable connections from any source.

```sh
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow=".*" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```

Save and close the modified files.

To enact these changes, restart the Tomcat service:

```sh
sudo systemctl restart tomcat
```

This will implement the new configurations and enable access to Tomcat's manager web app from remote systems.

### Step 4: Download and Install GeoServer

To integrate GeoServer with Apache Tomcat, begin by obtaining the GeoServer .war file. Visit the official GeoServer website at https://geoserver.org/ and navigate to the download page https://geoserver.org/download/. GeoServer offers two versions: Stable and Maintenance. The stable version is the latest release, while the maintenance version, though an earlier release, is maintained and officially supported for a specific duration. For this installation, we'll opt for the stable version.

Upon clicking the stable version link, you'll be redirected to https://geoserver.org/release/stable/. Right-click on the Web Archive under the Packages section and copy the link.

Now, open your Linux terminal and navigate to the temporary folder by typing cd /tmp. In this directory, use wget to download the GeoServer .war file by pasting the previously copied link.

#### Step 4.1: Download GeoServer

```sh
cd /tmp
wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.24.1/geoserver-2.24.1-war.zip
```

Next, unzip the downloaded file into the Tomcat webapps directory using the following commands:

```sh
sudo apt-get install unzip
sudo unzip geoserver-2.24.1-war.zip
```

#### Step 4.2: Install GeoServer in Tomcat

To integrate GeoServer with Tomcat, execute the following command to move GeoServer inside the Tomcat webapps directory:

```sh
mv geoserver.war /opt/tomcat/webapps/
```

This action concludes the installation process.

>Open in web browser 
>http://server_domain_or_IP:8080/geoserver

Upon opening this link, you'll arrive at the GeoServer welcome page. The default login credentials for GeoServer are **admin** as the username and **geoserver** as the password. Utilize these credentials to access the GeoServer admin panel for further configurations and operations.

### Step 5: Configuring Nginx Proxy for Tomcat with SSL

#### Step 5.1: Install Nginx

Begin by installing Nginx on your VPS:

```sh
sudo apt-get install nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

Next, install Certbot, a tool used for managing Let’s Encrypt certificates:

```sh
sudo apt-get install python3-certbot-nginx
```

To obtain a Let’s Encrypt SSL certificate, use the following Certbot commands:

For **_Subdomain_**

```sh
sudo certbot certonly --nginx -d subdomain.example.com
```

For **_Domain_**

```sh
sudo certbot certonly --nginx -d example.com
```

Upon successful certificate acquisition, Certbot automatically stores the certificate files. Note the certificate and key locations:

>Certificate is saved at: /etc/letsencrypt/live/subdomain.example.com/fullchain.pem 
>Key is saved at: /etc/letsencrypt/live/subdomain.example.com/privkey.pem

Allow both HTTP (80) and HTTPS (443) traffic through the firewall using Nginx Full:

```sh
sudo ufw allow 'Nginx Full'
```

#### Step 5.2: Create a new virtual host configuration file for Tomcat

Create and edit a new virtual host configuration file for Nginx:

```sh
sudo nano /etc/nginx/sites-available/geoserver
```

Insert the following configuration:

```sh
upstream tomcat {
    server 127.0.0.1:8080 fail_timeout=0;
}

server {
    listen 80;
    listen [::]:80;
    server_name subdomain.example.com;

    access_log /var/log/nginx/tomcat-access.log;
    error_log /var/log/nginx/tomcat-error.log;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl ipv6only=on;
    server_name subdomain.example.com;

    ssl_certificate /etc/letsencrypt/live/subdomain.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/subdomain.example.com/privkey.pem;

    location / {
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://tomcat/;
    }
}
```

#### Step 5.3: Auto renewal SSL Certificate

Configure automatic SSL certificate renewal by editing the crontab:

```sh
sudo crontab -e
```

Add the following line to the crontab file to perform a renewal check monthly:

```sh
0 0 1 * * certbot renew --nginx --quiet
```

### Step 6: Configure Cross-Origin Filter and Proxy Settings for Tomcat in GeoServer

Access the 'web.xml' file within the GeoServer application to enable its use in a Tomcat proxy:

```sh
sudo nano /opt/tomcat/webapps/geoserver/WEB-INF/web.xml
```

#### Step 6.1: Configure Proxy Base URL in GeoServer

Locate the following configuration and uncomment it to utilize the domain proxy:

```sh
<context-param>
  <param-name>PROXY_BASE_URL</param-name>
  <param-value>https://subdomain.example.com/geoserver</param-value>
</context-param>
```

Configure the allow list for [CSRF Protection](https://docs.geoserver.org/latest/en/user/security/webadmin/csrf.html) on Geoserver.

```sh
<context-param>
  <param-name>GEOSERVER_CSRF_WHITELIST</param-name>
  <param-value>subdomain.example.com</param-value>
</context-param>
```

#### Step 6.2: Enable Cross-Origin CorsFilter

Search for the following configuration and uncomment it to enable CORS in Catalina with Tomcat:

```sh
<!-- Uncomment following filter to enable CORS in Tomcat. Do not forget the second config block further down. -->
<filter>
    <filter-name>cross-origin</filter-name>
    <filter-class>org.apache.catalina.filters.CorsFilter</filter-class>
    <init-param>
        <param-name>cors.allowed.origins</param-name>
        <param-value>*</param-value>
    </init-param>
    <init-param>
        <param-name>cors.allowed.methods</param-name>
        <param-value>GET,POST,PUT,DELETE,HEAD,OPTIONS</param-value>
    </init-param>
    <init-param>
        <param-name>cors.allowed.headers</param-name>
        <param-value>*</param-value>
    </init-param>
</filter>
```

```sh
<!-- Uncomment following filter-mapping to enable CORS -->
<filter-mapping>
    <filter-name>cross-origin</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

Make sure to save the changes after uncommenting these configurations to apply the settings for the GeoServer application using Tomcat's proxy functionality.

### Step 7: Set Up GeoSpatial Database

#### Step 7.1: Install PostgreSQL 14 and PostGIS 3

Install PostgreSQL 14 and PostGIS 3 using the following commands:

```sh
sudo apt install postgis postgresql-14-postgis-3
psql --version
sudo systemctl status postgresql
```
#### Step 7.2: Create Database and User for the Service

Switch to the 'postgres' user:

```sh
sudo -i -u postgres
```

Using the 'postgres' user, create a user and a database:

```sh
createuser geo
createdb geodb -O geo
```

#### Step 7.2: Add the PostGIS Extension in the Database

Access the 'geodb' database:

```sh
psql -d geodb 
sudo -u geo psql geodb
```

Within the 'geodb' database, enable the PostGIS extension:

```sh
geodb=# CREATE EXTENSION postgis;
geodb=# CREATE EXTENSION postgis_topology;
geodb=# SELECT PostGIS_version();
```

Set a password for the 'geo' user in the Spatial Database and grant all privileges:

```sh
geodb=# ALTER USER geo WITH PASSWORD 'password';
geodb=# GRANT ALL PRIVILEGES ON DATABASE geodb TO geo;
geodb=# \q;
exit
```

#### Step 7.3: Expose the Spatial Database

Modify the PostgreSQL configuration file to allow connections from all origins:

```sh
sudo nano /etc/postgresql/14/main/postgresql.conf
```

Uncomment and modify the following line to listen on all IP addresses:

```sh
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'         # what IP address(es) to listen on;
```

Configure allowed hosts in the 'pg_hba.conf' file:

```sh
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

Add the following lines to allow connections to the 'geodb' database from any address:

```sh
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    geodb           geo             0.0.0.0/0               md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 md5
```

Allow incoming connections on port 5432 (PostgreSQL default port):

```sh
sudo ufw allow 5432/tcp
```

Restart the PostgreSQL service:

```sh
sudo systemctl restart postgresql
```

Finally, test the connection to the database from a local terminal:

```sh
psql -U userremoteconnexion -h server_ip_address_hosting_this_database -d mydatabase
```
