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

~~~
/etc/hosts  
=================================================
1. 127.0.0.1 localhost.localdomain localhost  
2. 192.0.2.0 hostname.example.com hostname  
~~~

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

![Install Step 1](https://raw.githubusercontent.com/bdsoftpro/email-server/master/1236-postfix_internetsite.png "Configure")  
![Install Step 2](https://github.com/bdsoftpro/email-server/blob/master/1237-postfix_systemmailname.png "Host Configure")  

This guide uses the following package versions:  
* Postfix 3.3.0
* Dovecot 2.2.33.2
* MySQL 14.14
## MySQL
The mail server’s virtual users and passwords are stored in a MySQL database. Dovecot and Postfix require this data. Follow the steps below to create the database tables for virtual users, domains and aliases:
1. Use the mysql_secure_installation tool to configure additional security options. This tool will ask if you want to set a new password for the MySQL root user, but you can skip that step:  
~~~
sudo mysql_secure_installation
~~~
Answer Y at the following prompts:
- Remove anonymous users?
- Disallow root login remotely?
- Remove test database and access to it?
- Reload privilege tables now?
2. Create a new database:
~~~
sudo mysqladmin -u root -p create mailserver
~~~
3. Log in to MySQL:
~~~
sudo mysql -u root -p
~~~
4. Create the MySQL user and grant the new user permissions over the database. Replace `mailuserpass` with a secure password:
~~~
GRANT SELECT ON mailserver.* TO 'mailuser'@'127.0.0.1' IDENTIFIED BY 'mailuserpass';
~~~
5. Flush the MySQL privileges to apply the change:
~~~
FLUSH PRIVILEGES;
~~~
6. Switch to the new `mailsever` database:
~~~
USE mailserver;
~~~
7. Create a table for the domains that will receive mail on the Linode:
~~~
CREATE TABLE `virtual_domains` (
  `id` int(11) NOT NULL auto_increment,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
~~~
8. Create a table for all of the email addresses and passwords:
~~~
CREATE TABLE `virtual_users` (
  `id` int(11) NOT NULL auto_increment,
  `domain_id` int(11) NOT NULL,
  `password` varchar(106) NOT NULL,
  `email` varchar(100) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `email` (`email`),
  FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
~~~~
9. Create a table for the email aliases:
~~~
CREATE TABLE `virtual_aliases` (
  `id` int(11) NOT NULL auto_increment,
  `domain_id` int(11) NOT NULL,
  `source` varchar(100) NOT NULL,
  `destination` varchar(100) NOT NULL,
  PRIMARY KEY (`id`),
  FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
~~~
## Adding Data
Now that the database and tables have been created, add some data to MySQL.  
1. Add the domains to the `virtual_domains` table. Replace the values for `example.com` and `hostname` with your own settings:
~~~
INSERT INTO `mailserver`.`virtual_domains`
  (`id` ,`name`)
VALUES
  ('1', 'example.com'),
  ('2', 'hostname.example.com'),
  ('3', 'hostname'),
  ('4', 'localhost.example.com');
~~~
>**Note**  
>Note which `id` corresponds to which domain, the `id` value is necessary for the next two steps.
