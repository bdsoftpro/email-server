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
| 
1. 127.0.0.1 localhost.localdomain localhost  
192.0.2.0 hostname.example.com hostname
|

