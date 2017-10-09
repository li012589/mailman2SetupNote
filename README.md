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
sudo mkdir /var/www/lists
sudo a2ensite mailman.conf
sudo /etc/init.d/apache2 restart
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

## Transports In /etc/postfix/transport

Alternatively, you can use **_mysql_**,  but for simplicity we use the file _/etc/postfix/transport_ instead.

```
sudo postconf -e 'transport_maps = hash:/etc/postfix/transport'
```

Open _/etc/postfix/transport_:

```
sudo emacs /etc/postfix/transport
```

add this line:

```
lists.example.com      mailman:
```

Remember to exchange _lists.example.com_ with your domain name. Then

```
sudo postmap -v /etc/postfix/transport
sudo /etc/init.d/postfix restart
```

## Create the mailman mailing list

Before we can start to use **_Mailman_**, we must create a mailing list called _mailman_; this is obligatory - without it **_Mailman_** won't start:

```
sudo newlist --urlhost=lists.example.com --emailhost=lists.example.com mailman
```

In most cases the `—urlhos`t and `—emailhost` switches are not necessary because our vhost is already named _lists.example.com_, and we also have it in _/etc/mailman/mm_cfg.py_ (`DEFAULT_EMAIL_HOST` and `DEFAULT_URL_HOST`), but if you want to go sure that **_Mailman_** uses the correct hostname, use these switches.

In case for mistake, your can remove list as root user:

```
sudo rmlist <the_list_to_remove>
```

The followings will be prompted:

```
Enter the email of the person running the list: <-- specify the list administrator email address, e.g. sales@example.com
Initial mailman password: <-- mailman_password
To finish creating your mailing list, you must edit your /etc/aliases (or
equivalent) file by adding the following lines, and possibly running the
`newaliases' program:

## mailman mailing list
mailman:              "|/var/lib/mailman/mail/mailman post mailman"
mailman-admin:        "|/var/lib/mailman/mail/mailman admin mailman"
mailman-bounces:      "|/var/lib/mailman/mail/mailman bounces mailman"
mailman-confirm:      "|/var/lib/mailman/mail/mailman confirm mailman"
mailman-join:         "|/var/lib/mailman/mail/mailman join mailman"
mailman-leave:        "|/var/lib/mailman/mail/mailman leave mailman"
mailman-owner:        "|/var/lib/mailman/mail/mailman owner mailman"
mailman-request:      "|/var/lib/mailman/mail/mailman request mailman"
mailman-subscribe:    "|/var/lib/mailman/mail/mailman subscribe mailman"
mailman-unsubscribe:  "|/var/lib/mailman/mail/mailman unsubscribe mailman"

Hit enter to notify mailman owner... <-- ENTER
```

Now, as suggested edit _/etc/aliases_

```
sudo emacs /etc/aliases
```

add these lines:

```
## mailman mailing list
mailman:              "|/var/lib/mailman/mail/mailman post mailman"
mailman-admin:        "|/var/lib/mailman/mail/mailman admin mailman"
mailman-bounces:      "|/var/lib/mailman/mail/mailman bounces mailman"
mailman-confirm:      "|/var/lib/mailman/mail/mailman confirm mailman"
mailman-join:         "|/var/lib/mailman/mail/mailman join mailman"
mailman-leave:        "|/var/lib/mailman/mail/mailman leave mailman"
mailman-owner:        "|/var/lib/mailman/mail/mailman owner mailman"
mailman-request:      "|/var/lib/mailman/mail/mailman request mailman"
mailman-subscribe:    "|/var/lib/mailman/mail/mailman subscribe mailman"
mailman-unsubscribe:  "|/var/lib/mailman/mail/mailman unsubscribe mailman"
```

Whenever you modify _/etc/aliases_, you need to run

```
sudo newaliases
sudo /etc/init.d/postfix restart
```

Now we can finally start **_Mailman_**:

```
sudo /etc/init.d/mailman start
```

Then the **_Mailman_** server is up and running.

Now configure password to create new list on web page:

```
sudo mmsitepass <mailman_password>
```

## Web administration

* The admin interface can be reached on _http://lists.example.com/admin/mailman_
  * you can log in use user name and password specified at creating _mailman_ list
* New lists can create at _http://lists.example.com/create_
  * password are created with `mmsitpass` as mentioned above.
  * notice that owner will receive a email telling you to edit _\etc\aliases_, refer to next section for more detailed instructions.
* General information about lists hosted can be viewed at both _http://lists.example.com/listinfo_ or just _http://lists.example.com/_ (as configured above at apache2 section)

## Command line administration

To add new lists, run:

```
newlist --urlhost=lists.example.com --emailhost=lists.example.com <list_name>
```

The following will be prompted:

```
Enter the email of the person running the list: <-- onwer's email address
Initial testlist2 password: <-- onwer's password
To finish creating your mailing list, you must edit your /etc/aliases (or
equivalent) file by adding the following lines, and possibly running the
`newaliases' program:

## testlist2 mailing list
testlist2:              "|/var/lib/mailman/mail/mailman post <list_name>"
testlist2-admin:        "|/var/lib/mailman/mail/mailman admin <list_name>"
testlist2-bounces:      "|/var/lib/mailman/mail/mailman bounces <list_name>"
testlist2-confirm:      "|/var/lib/mailman/mail/mailman confirm <list_name>"
testlist2-join:         "|/var/lib/mailman/mail/mailman join <list_name>"
testlist2-leave:        "|/var/lib/mailman/mail/mailman leave <list_name>"
testlist2-owner:        "|/var/lib/mailman/mail/mailman owner <list_name>"
testlist2-request:      "|/var/lib/mailman/mail/mailman request <list_name>"
testlist2-subscribe:    "|/var/lib/mailman/mail/mailman subscribe <list_name>"
testlist2-unsubscribe:  "|/var/lib/mailman/mail/mailman unsubscribe <list_name>"

Hit enter to notify testlist2 owner... <-- ENTER
```

Edit _/etc/aliases_ as instructed

```
sudo emacs /etc/aliases
```

add these lines as mentioned above (DON'T copy it here, different from run to run). Then

```
sudo newaliases
sudo /etc/init.d/postfix restart
```

## Reference

1. https://www.howtoforge.com/how-to-install-and-configure-mailman-with-postfix-on-debian-squeeze
2. https://help.ubuntu.com/community/Mailman
3. http://www.gnu.org/s/mailman/index.html