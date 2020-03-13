# Email with Postfix, Dovecot, and MySQL
Email server setup details, that describes brllow:  

## Before You Begin
1. Set up the Linode as specified in the Getting Started and Securing Your Server guides.  
2. Verify that the iptables firewall is not blocking any of the standard mail ports (`25`, `465`, `587`, `110`, `995`, `143`, and `993`). If using a different form of firewall, confirm that it is not blocking any of the needed ports.  
3. Review the concepts in the Running a Mail Server guide.  

## Configure DNS
When you’re ready to update the DNS and start sending mail to the server, edit the domain’s MX record so that it points to the Linode’s domain or IP address, similar to the example below:  
~~~
example.com A 10 12.34.56.78  
example.com MX 10 example.com  
mail.example.com MX 10 example.com  
~~~
Make sure that the MX record is changed for all domains and subdomains that might receive email. If setting up a brand new domain, these steps can be performed prior to configuring the mail server. When using Linode’s DNS Manager, create an MX record that points to the desired domain or subdomain, and then create an A record for that domain or subdomain, which points to the correct IP address.  
## Update Hosts File
Verify that the `hosts file` contains a line for the Linode’s public IP address and is associated with the **Fully Qualified Domain Name** (FQDN). In the example below, `192.0.2.0` is the public IP address, `hostname` is the local hostname, and `hostname.example.com` is the FQDN.  

| /etc/hosts	|
| ---      |
| 1. 127.0.0.1 localhost.localdomain localhost  |
| 2. 192.0.2.0 hostname.example.com hostname	  |

## Install SSL Certificate
You will need to install a SSL certificate on your mail server prior to completing the Dovecot configuration steps. The SSL certificate will authenticate the identity of the mail server to users and encrypt the transmitted data between the user’s mail client and the mail server. Follow our guide to Install an SSL certificate with Certbot.  
  
Make a note of the certificate and key locations on the Linode. You will need the path to each during the Dovecot configuration steps.  

## Install Packages
1. Log in to your Linode via SSH. Replace 192.0.2.0 with your IP address:
~~~
ssh username@192.0.2.0
~~~
2. Install the required packages:
~~~
sudo apt-get install postfix postfix-mysql dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd dovecot-mysql mysql-server
~~~
You will not be prompted to enter a password for the root MySQL user for recent versions of MySQL. This is because on Debian and Ubuntu, MySQL now uses either the `unix_socket` or `auth_socket` authorization plugin by default. This authorization scheme allows you to log in to the database’s root user as long as you are connecting from the Linux root user on localhost.  
  
When prompted, select **Internet Site** as the type of mail server the Postfix installer should configure. The System Mail Name should be the FQDN.  

![Install Step 1](https://www.linode.com/docs/email/postfix/email-with-postfix-dovecot-and-mysql/1236-postfix_internetsite.png "Configure")  
![Install Step 2](https://www.linode.com/docs/email/postfix/email-with-postfix-dovecot-and-mysql/1237-postfix_systemmailname.png "Host Configure")
