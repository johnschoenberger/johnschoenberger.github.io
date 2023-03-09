---
title: LCM Darksite Web Server with SSL
date: 2023-03-09 17:05:02 -400
categories: [homelab, hardware, nutanix, lcm]
tags: [servers, hardware, network, nutanix, rack, lcm]
---

# LCM Darksite Web Server with SSL
Building a LCM Darksite webserver is pretty simple using Linux. It can be done on Windows Server with IIS but I highly recommend using your favorite flavor of Linux. The following tutorial will be using Rocky Linux...
This guide is based of the [Life Cycle Manager Dark Site Guide LCM 2.4 - Last updated: 2023-02-28](
https://portal.nutanix.com/page/documents/details?targetId=Life-Cycle-Manager-Dark-Site-Guide-v2_4:Life-Cycle-Manager-Dark-Site-Guide-v2_4)

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

8. Restart httpd.
``` bash
sudo systemctl restart httpd
```

9. Enable Apache to start when the system starts.
``` bash
sudo systemctl enable httpd
```	

10. Open a browser and enter the static IP address assigned to your web server in the search bar.
You should receieve the "Testing 123..." Apache page

11. Create a directory called release under the document root /var/www/html.
``` bash
sudo mkdir -p -m 755 /var/www/html/release
cd /var/www/html/release
```	

12. Optional: Set Selinux context to folder and files
``` bash
sudo chmod -R 755 /var/www/html/release
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/html/release(/.*)?"
sudo restorecon -R -v /var/www/html/release
```

## Enable SSL/Install certs
---
1. Install mod_ssl
``` bash 
sudo yum install mod_ssl -y
```

2. Generate a CSR and Private Key with OpenSSL
``` bash
sudo openssl req -new -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/server.key -out /etc/pki/tls/private/server.csr
```
You will then be prompted for input regarding your CSR:

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

* Now edit the secure web server configuration file and locate/modify the directives as shown below.

``` bash
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLCertificateFile /etc/pki/tls/certs/certificate.crt
SSLCertificateKeyFile /etc/pki/tls/private/server.key
SSLCACertificateFile /etc/pki/tls/certs/intermediate.crt
```
Save & Exit file

7. Create a LCM virtual host file
``` bash
sudo vi /etc/httpd/conf.d/lcmrelease.conf
```

``` bash
<VirtualHost *:443>
    ServerName www.yourdomain.com
    ServerAlias yourdomain.com
    DocumentRoot /var/www/html/
    ErrorLog /var/www/yourdomain.com/error.log
    CustomLog /var/www/yourdomain.com/requests.log combined
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/certificate.crt
    SSLCertificateKeyFile /etc/pki/tls/private/server.key
    SSLCACertificateFile /etc/pki/tls/certs/intermediate.crt
</VirtualHost>
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

## Updating PE/PC cert trust chains
---
The following is based on [Nutanix KB-5090](https://portal.nutanix.com/kb/5090)
 
 Pre-Reqs:
 * Use "PEM" format.
 * Copy the CA Chain to one of the CVMs/PCVMs. In the example below, we put the file in /home/nutanix/tmp/custom_ssl/cachain.crt.
 
 Execute the following commands to install the certificate on all CVMs and PCVMs:
1. Create folder to store the CA chain file
 ``` bash
allssh 'mkdir /home/nutanix/tmp/custom_ssl/'
 ```
2. Copy/Paste contents of CA Chain into /home/nutanix/tmp/custom_ssl/cachain.crt
``` bash
vi /home/nutanix/tmp/custom_ssl/cachain.crt
```
Save & Exit File
3. Copy file to all CVMs
``` bash
allssh 'rsync -avh <cvm_ip>:/home/nutanix/tmp/custom_ssl/cachain.crt /home/nutanix/tmp/custom_ssl/'
```
4. To allow trusted certificates to persist after upgrades
``` bash
pki_ca_certs_manager set -p /home/nutanix/tmp/custom_ssl/cachain.crt
```

> As of AOS 6.1, 6.0.2, pc.2022.1, 5.20.2, 5.20.3 or newer, the certificates can be persisted in the CVM database via the following command once relevant certs are copied to a directory on the CVM/PCVM.
{: .prompt-info }

## Updating MSP controller cert trust chains
---
The following is based on [Nutanix KB-14240](https://portal.nutanix.com/kb/14240)
> This is a internal KB; please open a ticket & work with SRE before attempting these steps.
{: .prompt-warning }

### Description: 
Customers using LCM https darksite web server might notice MSP upgrades fail with the following error in msp_controller.out file

``` bash
2023-01-27 07:01:50,102Z INFO helper.py:145 (xx, update, 2ca52cfc-11eb-422a-677f-9389075e89ab) ERROR: Validate precheck failed.airgap server validation failure: Host Image not found at AirGap Server: Get "https://xxx/release//builds/msp-builds/host-images/msp-goldimages-2_4_2-3.json": x509: certificate signed by unknown authority Status Code: 500
```

The issue happens when customer uses self signed certificate for the LCM darksite web server which is not trusted by MSP controller.

### Solution:
Prior proceeding with the workaround below ensure the LCM darksite web server url is https based and customer is using untrusted certificate. This can be easily verified by opening the web server url in browser and check the certificate status. 

Once confirmed the customer is using a https server with untrusted certificate follow the solution below to fix the issue.

1. Follow steps above for [Nutanix KB-5090](https://portal.nutanix.com/kb/5090)
2. Verify Prism Central is successfully able to reach the LCM darksite web server url without failing with certificate untrusted error.
``` bash
allssh curl <lcm_darksite_url>
```
Replace <lcm_darksite_url> with the url from LCM Settings webpage.
Once the above command works without certificate errors, proceed with the next steps
3. Take a backup of the msp-controller-config.json file by running the command below,
``` bash
allssh "cp /home/docker/msp_controller/bootstrap/msp-controller-config.json ~/tmp/msp-controller-config.json_backup"
```
4. Run the below command to replace msp-controller-config.json with a bind mount
``` bash
allssh "sed -i '/^\s*\"binds\".*/a \     \ \"/etc/pki:/etc/pki\",'  /home/docker/msp_controller/bootstrap/msp-controller-config.json"
```
5. Restart msp controller service.
``` bash
allssh genesis stop msp-controller; cluster start
```
6. Verify curl can be done inside msp-controller without certificate errors,
``` bash
allssh 'docker exec  msp-controller bash -c "curl <lcm_darksite_url>"'
```
Replace <lcm_darksite_url> with the url from LCM Settings webpage.
Restart the LCM upgrade for MSP controller.