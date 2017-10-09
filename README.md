# Mailman2 Setup 

In this tutorial I will configure a virtual host with the hostname **_lists.example.com_** where I will install Mailman. **_lists.example.com_** is also the right part of the mailing list email addresses that will be configured in Mailman, i.e., mails to a mailing list will have to be sent to the address **_\<listname\>@lists.example.com_**.

In this tutorial, **_apache2 postfix_** and **_mailman_** will be used.   

## Web server setup

The default web server is **_apache2_**, alternatively nginx is ok. 

` sudo apt install apache2`

You will see the following questions/messages:

```
Languages to support: <-- Chose your languages
Missing site list
Mailman needs a so-called "site list", which is the list from which password reminders and such are sent out from. This list needs to be created before mailman will start. To create the list, run "newlist mailman" and follow the instructions on-screen. Note that you also need to start mailman after that, using /etc/init.d/mailman start. <-- Ok
```

Then set virtual host for **_mailman_**, it comes with an Apache configuration file, _/etc/mailman/apache.conf_

` sudo cp /etc/mailman/apache.conf /etc/apache2/sites-available/mailman.conf`

Then edit it to fit your DNS

` sudo emacs /etc/apache2/sites-available/mailman.conf`

Append the following to it

```
<VirtualHost *:80>
ServerName lists.example.com <-- exchange this with your domain name
DocumentRoot /var/www/lists
ErrorLog /var/log/apache2/lists-error.log
CustomLog /var/log/apache2/lists-access.log combined

<Directory /var/lib/mailman/archives/>
    Options FollowSymLinks
    AllowOverride None
</Directory>

Alias /pipermail/ /var/lib/mailman/archives/public/
Alias /images/mailman/ /usr/share/images/mailman/
ScriptAlias /admin /usr/lib/cgi-bin/mailman/admin
ScriptAlias /admindb /usr/lib/cgi-bin/mailman/admindb
ScriptAlias /confirm /usr/lib/cgi-bin/mailman/confirm
ScriptAlias /create /usr/lib/cgi-bin/mailman/create
ScriptAlias /edithtml /usr/lib/cgi-bin/mailman/edithtml
ScriptAlias /listinfo /usr/lib/cgi-bin/mailman/listinfo
ScriptAlias /options /usr/lib/cgi-bin/mailman/options
ScriptAlias /private /usr/lib/cgi-bin/mailman/private
ScriptAlias /rmlist /usr/lib/cgi-bin/mailman/rmlist
ScriptAlias /roster /usr/lib/cgi-bin/mailman/roster
ScriptAlias /subscribe /usr/lib/cgi-bin/mailman/subscribe
ScriptAlias /mailman/ /usr/lib/cgi-bin/mailman/
ScriptAlias / /usr/lib/cgi-bin/mailman/listinfo
</VirtualHost>
```

The second to last line `ScriptAlias / /usr/lib/cgi-bin/mailman/listinfo` is optional; it makes that when you go to _http://lists.example.com/_, you will be redirected to _http://lists.example.com/listinfo_. This makes sense if you don't have any files to serve in the document root _/var/www/lists_.

Next create the empty document root _/var/www/lists_ (placeholder), enable the _lists.example.com_ vhost configuration and restart Apache:

```
mkdir /var/www/lists
a2ensite mailman.conf
/etc/init.d/apache2 restart
```

Also, change domain name at _/etc/mailman/mm_cfg.py_ by

```
sudo emacs /etc/mailman/mm_cfg.py
```

change these lines:

```
DEFAULT_URL_PATTERN = 'http://%s/'

DEFAULT_EMAIL_HOST = 'lists.example.com' <-- exchange this with your domain name

DEFAULT_URL_HOST = 'lists.example.com' <-- exchange this with your domain name
```

## Mail server setup

First install **_postfix_**

```
sudo apt install postfix
```

In installling process, just use default settings and enter the domain name when asked. Then

```
sudo postconf -e 'relay_domains = lists.example.com' <-- exchange this with your domain name
sudo postconf -e 'mailman_destination_recipient_limit = 1'
```

Then make sure you have the right lines in _/etc/postfix/master.cf_, by

```
cat /etc/postfix/master.cf
```

There should be lines like

```
mailman   unix  -       n       n       -       -       pipe
  flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
  ${nexthop} ${user}
```

