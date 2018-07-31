---
published: true
---
#### A quick tutorial on how to configure the AutoMX WSGI script with a MySQL database like the ones PostfixAdmin and ViMbAdmin use.

I recently moved my stuff, including this site, to a new server (thanks to [php-friends](https://php-friends.de)). I also had to set up a new full-blown mailserver with virtual mailboxes there (following [this](https://www.debinux.de/2015/05/mailserver-from-scratch-debian-8/) tutorial).  

To allow my users to connect to their email accounts by only inputting their email address and password, I installed AutoMX (https://automx.org) from the official debian package. The tutorial uses [ViMbAdmin](http://www.vimbadmin.net/), which, like PostfixAdmin, uses a MySQL database to represent users.  
When the email client, e.g. Thunderbird, requests account data from the domain in the *domain-part* in the address (whatever is behind the *@*) trough either the open *Autoconfig* protocol or, in the case of Outlook, the *Autodiscover* protocol, this request is proxied to the AutoMX Python wsgi script, which gets the account data from the database and then outputs the XML-formatted info to the client.  
So far in theory. Now on to the configuration.  

Edit `/etc/automx.conf`:  

1. Change the main `automx` section to represent your host: 

  ~~~
  [automx]
  provider = mail.domain.com	   
  domains = *
  ~~~   

  Where `mail.domain.com` is replaced by your actual mailserver\'s hostname. `domains` is set to `* ` because we want to serve all domains from the database.  

2. Now we want to configure MySQL in the `global` backend section.  
  For `ViMbAdmin` :   

  ~~~
  [global]
  backend = sql
  action = settings

  host = mysql://vimbadmin:password@localhost/vimbadmin
  query = SELECT mailbox.name AS display_name, domain.domain, mailbox.username AS email FROM mailbox, domain WHERE mailbox.Domain_id = domain.id AND username=\'%s\' 

  result_attrs = display_name, email, domain 
  ~~~

  Or for `Postfixadmin`:

  ~~~
  [global]
  backend = sql
  action = settings

  host = mysql://postfix:password@localhost/postfix
  query = SELECT name AS display_name, domain, username AS email FROM mailbox WHE$
  result_attrs = display_name, email, domain

  account_name = ${display_name}
  ~~~

  Of course, replace `password` with your actual database user password.

  The other options are now pretty straight-forward since you can use any of the `result_attrs`as a variable in the configuration like `${display_name}`. Here is an example for the rest of the `global` section:

  ~~~
  account_name = ${display_name}
  smtp = yes 
  smtp_server = mail.${domain}
  smtp_port = 587 
  smtp_encryption = starttls
  smtp_auth = plaintext
  smtp_auth_identity = ${email}
  smtp_refresh_ttl = 6 
  smtp_default = yes 

  smtp_author = ${display_name}

  imap = yes 
  imap_server = $mail.{domain}
  imap_port = 143 
  imap_encryption = starttls
  imap_auth = plaintext
  imap_auth_identity = ${email}
  imap_refresh_ttl = 6 

  pop = no  
  ~~~

  There are no specific sections for domains needed, so you can happily comment out the section with `[example.com]`.
  
3. AutoMX itself should work now, but we still need to configure the webserver to proxy the request. AutoMX comes with example configuration files for both Apache and NGINX (located in */usr/share/doc/automx/examples* ).  
  I have adjusted the Apache example like this (*/etc/apache2/sites-available/automx.conf*):  

  ~~~
  <VirtualHost *:80>
	  ServerName autoconfig.example.com
	  ServerAlias autoconfig.*
	  DocumentRoot /badurl
		<IfModule mod_wsgi.c>
				WSGIScriptAliasMatch \\
						(?i)^/.+/(autodiscover|config-v1.1).xml \\
			/usr/lib/python2.7/dist-packages/automx_wsgi.py
				<Directory \"/usr/lib/python2.7/dist-packages\">
			Require all granted
				</Directory>
		</IfModule>
  </VirtualHost>
  
  <VirtualHost *:443>
	  ServerName autodiscover.example.com
	  ServerAlias autodiscover.*
	  DocumentRoot /badurl
	  SSLEngine On
	  SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem	
	  SSLCertificateKeyFile  /etc/letsencrypt/live/example.com/privkey.pem
		<IfModule mod_wsgi.c>
				WSGIScriptAliasMatch \\
						(?i)^/.+/(autodiscover|config-v1.1).xml \\
			/usr/lib/python2.7/dist-packages/automx_wsgi.py
				WSGIScriptAlias \\
						/mobileconfig \\
			/usr/lib/python2.7/dist-packages/automx_wsgi.py
				<Directory \"/usr/lib/python2.7/dist-packages\">
			Require all granted
				</Directory>
		</IfModule>
  </VirtualHost>
  ~~~
Of course replace all instances of `example.com`with your actual domain and correctly set the paths for the SSL certs.
  
4.  The last step is to enable the `automx.conf` with `a2ensite automx` and reload Apache with `service apache2 reload` or `systemctl reload apache2`.

Now you are all done and can test your configuration with `automx-test someuser@example.com`. If everything works out fine, well done! If you still have any questions (including setting up AutoMX on FreeBSD), don't hesitate to leave me a comment!  
Happy configuring!

*Note: This article was transferred over from my old website in June, 2018*
