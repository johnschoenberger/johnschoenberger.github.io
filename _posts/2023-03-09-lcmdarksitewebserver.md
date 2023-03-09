---
title: LCM Darksite Web Server with SSL
date: 2023-03-09 15:14:12 -400
categories: [homelab, hardware, lcm]
tags: [servers, hardware, network, nutanix, rack, lcm]
---

# LCM Darksite Web Server with SSL
Building a LCM Darksite webserver is pretty simple using Linux. It can be done on Windows Server with IIS but I highly recommend using your favorite flavor of Linux. The following tutorial will be using Rocky Linux...
This guide is based of the Life Cycle Manager Dark Site Guide (LCM 2.4) - Last updated: 2023-02-28
https://portal.nutanix.com/page/documents/details?targetId=Life-Cycle-Manager-Dark-Site-Guide-v2_4:Life-Cycle-Manager-Dark-Site-Guide-v2_4

* Build a Rocky Linux VM using minimal install
* Set static IP address

## Procedure
---
1. Update the yum repository on your Linux server.
``` bash
sudo yum update -y
```

2. Install httpd.
``` bash
sudo yum install httpd -y
```

3. Check that SELinux is enabled.
``` bash
getenforce
```						
* If SELinux is Enabled; returns the string Enforcing.
* If SELinux is Disabled; returns the string Disabled.

4. To enable or disable SELinux, use the following command.
``` bash
sudo setenforce 1|0
```

5. Set the firewall to allow Apache access.
``` bash
sudo firewall-cmd --permanent --zone=public --add-service=http
```

6. Reload the firewall to make your changes take effect.
``` bash
sudo firewall-cmd --reload
```

7. In a text editor, open the main server configuration file at /etc/httpd/conf/httpd.conf.
``` bash
sudo vi /etc/httpd/conf/httpd.conf
```

Make sure that httpd.conf contains the following lines. If they are commented out, uncomment them.
``` bash
ServerRoot "/etc/httpd"
DocumentRoot "/var/www/html" #All the documents will be served under this directory hierarchy
ServerName your-web-server-IP-address|Fully qualified domain name #if nameservice is available
Listen IP-address:port
```
Save and Exit file

9. Restart httpd.
``` bash
sudo systemctl restart httpd
```

10. Enable Apache to start when the system starts.
``` bash
sudo systemctl enable httpd
```	

11. Open a browser and enter the static IP address assigned to your web server in the search bar.
You should receieve the Testing 123... Apache page

12. Create a directory called release under the document root /var/www/html.
``` bash
sudo mkdir -p -m 755 /var/www/html/release
cd /var/www/html/release
```	

13. Optional: Set Selinux context to folder and files
``` bash
sudo chmod -R 755 /var/www/html/release
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/html/release(/.*)?"
sudo restorecon -R -v /var/www/html/release
```

## Enable SSL/Install certs
---
1. Install mod_ssl
``` bash 
sudo yum install mod_ssl
```

2. Generate a CSR and Private Key with OpenSSL
``` bash
sudo openssl req -new -newkey rsa:4096 -nodes -keyout /etc/pki/tls/private/server.key -out /etc/pki/tls/private/server.csr
```
You will then get a prompt asking you to input the following details regarding your CSR:-

* Country Name (2 letter code) [AU]: Type in the 2 letter abbreviation for your country.
* State or Province Name (full name) [Some-State]: Full name of the state
* Organization Name (eg, company) [Internet Widgits Pty Ltd]:Locality Name (eg, city) []: Complete name of the city, no abbreviations
* Organization Name (eg, company) [Internet Widgits Pty Ltd]: If you are a business; Enter your legal entity name. If you're not a business, any value entered will not be used in your certificate.
* Organizational Unit Name (eg, section) []: If you are a business; Write the appropriate division of your company. It is best to use something generic such as "IT".
* Common Name (e.g. server FQDN or YOUR name) []: Enter your domain name
* Email Address []: Enter your email address

After you hit Enter, your Private Key and CSR should be saved successfully in the default ssl directory.

3. View CSR to submit to PKI server
``` bash
sudo cat /etc/pki/tls/private/server.csr
```

4. Retrieve your new certificate from PKI server and copy/paste or scp into the following file
``` bash
sudo vi /etc/pki/tls/certs/certificate.crt
```

5. Upload Intermediate/Root/CA chains
``` bash
sudo vi /etc/pki/tls/certs/intermediate.crt
```

6. Configure appropriate httpd SSL parameters
``` bash
sudo vi /etc/httpd/conf.d/ssl.conf
```

Now edit the secure web server configuration file and locate/modify the directives as shown below.

``` bash
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLCertificateFile /etc/pki/tls/certs/certificate.crt
SSLCertificateKeyFile /etc/pki/tls/private/server.key
SSLCACertificateFile /etc/pki/tls/certs/intermediate.crt
```
Save & Exit file

Verify changes with
``` bash
 httpd -t
```

7. Configure the httpd Virtual Host
``` bash
sudo vi /etc/httpd/conf/httpd.conf
```
``` bash
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/certificate.crt
SSLCertificateKeyFile /etc/pki/tls/private/server.key
SSLCACertificateFile /etc/pki/tls/certs/intermediate.crt
servername www.yourdomain.com
Documentroot /var/www/html/release
```

Change yourdomain.com to your domain name. Save & Exit file

8. Add https to firewall
``` bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

9. Enable/Restart httpd service
``` bash
sudo systemctl enable httpd.service && sudo systemctl restart httpd.service
```

