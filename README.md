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

2. Add email addresses to the `virtual_users` table. The `domain_id` value references the `virtual_domain` table’s `id` value. Replace the email address values with the addresses that you wish to configure on the mailserver. Replace the `password` values with strong passwords.
~~~
INSERT INTO `mailserver`.`virtual_users`
  (`id`, `domain_id`, `password` , `email`)
VALUES
  ('1', '1', ENCRYPT('password', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email1@example.com'),
  ('2', '1', ENCRYPT('password', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'email2@example.com');
~~~
3. An email alias will forward all email from one email address to another. To set up an email alias, add it to the `virtual_aliases` table:
~~~
INSERT INTO `mailserver`.`virtual_aliases`
  (`id`, `domain_id`, `source`, `destination`)
VALUES
  ('1', '1', 'alias@example.com', 'email1@example.com');
~~~
## Testing
In the previous section, data was added to the MySQL `mailserver` database. The steps below will test that the data has been stored and can be retrieved.
1. Log in to MySQL:
~~~
sudo mysql -u root
~~~
2. Check the contents of the `virtual_domains` table:
~~~
SELECT * FROM mailserver.virtual_domains;
~~~
3. Verify that you see the following output:
~~~
+----+-----------------------+
| id | name                  |
+----+-----------------------+
|  1 | example.com           |
|  2 | hostname.example.com  |
|  3 | hostname              |
|  4 | localhost.example.com |
+----+-----------------------+
4 rows in set (0.00 sec)
~~~
4. Check the `virtual_users` table:
~~~
SELECT * FROM mailserver.virtual_users;
~~~
5. Verify the following output, the hashed passwords are longer than they appear below:
~~~
+----+-----------+-------------------------------------+--------------------+
| id | domain_id | password                            | email              |
+----+-----------+-------------------------------------+--------------------+
|  1 |         1 | $6$574ef443973a5529c20616ab7c6828f7 | email1@example.com |
|  2 |         1 | $6$030fa94bcfc6554023a9aad90a8c9ca1 | email2@example.com |
+----+-----------+-------------------------------------+--------------------+
2 rows in set (0.01 sec)
~~~
6. Check the `virtual_aliases` table:
~~~
SELECT * FROM mailserver.virtual_aliases;
~~~
7. Verify the following output:
~~~
+----+-----------+-------------------+--------------------+
| id | domain_id | source            | destination        |
+----+-----------+-------------------+--------------------+
|  1 |         1 | alias@example.com | email1@example.com |
+----+-----------+-------------------+--------------------+
1 row in set (0.00 sec)
~~~
8. If everything outputs as expected, exit MySQL:
~~~
exit
~~~
## Postfix
Postfix is a Mail Transfer Agent (MTA) that relays mail between the Linode and the internet. It is highly configurable, allowing for great flexibility. This guide maintains many of Posfix’s default configuration values.
## Configuration File Settings
The `main.cf` file is the primary configuration file used by Postfix.
1. Make a copy of the default Postfix configuration file in case you need to revert to the default configuration:
~~~
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.orig
~~~
2. Edit the `/etc/postfix/main.cf` file to match the example configurations. Replace occurrences of `example.com` with your domain name:
~~~
/etc/postfix/main.cf
==========================================================================================
# See /usr/share/postfix/main.cf.dist for a commented, more complete version

# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# TLS parameters
smtpd_tls_cert_file=/etc/letsencrypt/live/example.com/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/example.com/privkey.pem
smtpd_use_tls=yes
smtpd_tls_auth_only = yes
smtp_tls_security_level = may
smtpd_tls_security_level = may
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

# Authentication
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

# Restrictions
smtpd_helo_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_invalid_helo_hostname,
        reject_non_fqdn_helo_hostname
smtpd_recipient_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_non_fqdn_recipient,
        reject_unknown_recipient_domain,
        reject_unlisted_recipient,
        reject_unauth_destination
smtpd_sender_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_non_fqdn_sender,
        reject_unknown_sender_domain
smtpd_relay_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        defer_unauth_destination

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

myhostname = example.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mydomain = example.com
myorigin = $mydomain
mydestination = localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

# Handing off local delivery to Dovecot's LMTP, and telling it where to store mail
virtual_transport = lmtp:unix:private/dovecot-lmtp

# Virtual domains, users, and aliases
virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
virtual_alias_maps = mysql:/etc/postfix/mysql-virtual-alias-maps.cf,
        mysql:/etc/postfix/mysql-virtual-email2email.cf

# Even more Restrictions and MTA params
disable_vrfy_command = yes
strict_rfc821_envelopes = yes
#smtpd_etrn_restrictions = reject
#smtpd_reject_unlisted_sender = yes
#smtpd_reject_unlisted_recipient = yes
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtp_always_send_ehlo = yes
#smtpd_hard_error_limit = 1
smtpd_timeout = 30s
smtp_helo_timeout = 15s
smtp_rcpt_timeout = 15s
smtpd_recipient_limit = 40
minimal_backoff_time = 180s
maximal_backoff_time = 3h

# Reply Rejection Codes
invalid_hostname_reject_code = 550
non_fqdn_reject_code = 550
unknown_address_reject_code = 550
unknown_client_reject_code = 550
unknown_hostname_reject_code = 550
unverified_recipient_reject_code = 550
unverified_sender_reject_code = 550
~~~
3. The `main.cf` file declares the location of `virtual_mailbox_domains`, `virtual_mailbox_maps`, and `virtual_alias_maps` files. These files contain the connection information for the MySQL lookup tables created in the MySQL section of this guide. Postfix will use this data to identify all domains, corresponding mailboxes, and valid users.
  Create the file for `virtual_mailbox_domains`. Replace the value for `password` with your database user’s password. If you used a different name for your `database` user and `dbname` replace those with your own values:
~~~
/etc/postfix/mysql-virtual-mailbox-domains.cf
====================================================================
user = mailuser
password = mailuserpass
hosts = 127.0.0.1
dbname = mailserver
query = SELECT 1 FROM virtual_domains WHERE name='%s'
~~~

