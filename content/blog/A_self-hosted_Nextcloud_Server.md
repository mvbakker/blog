---
title: "A Self-hosted Nextcloud server"
date: "2016-08-20"
draft: false
slug: self-hosted-nextcloud-server
author: "Michiel Bakker"
---

So I've been thinking about setting up my own cloud service for a while now. I think it is good to be a little selfhosted nowadays. Recently I bought an [UP-Board][upboard] and this little powerfull computer seemed just perfect for this project. In this post I will not walk you through all the steps specifically. However I will show you what manuals and instructions I used and what problems arose during the process and how I solved those. Also, being a beginner myself, I try to understand what is actually all happening and explain my findings to you. Probably I am not a 100% correct, but just trying to explain it to you helps me as well. Good luck and I hope you enjoy!

[upboard]: http://up-board.org

Server with a LAMP stack installed
-----
So a server is your computer, or in my case this small UP-Board. You can run Linux and attach a screen with a keyboard and mouse and it will just work as a regular desktop computer. However you can also run a linux-server on it. This has some major differences, the first being that this linux-server will not have a graphical user interface. So there is only a terminal showing the commandline. (Honestly, if you're interested in these kind of things I encourage you to become really handy with working in the commandline as soon as possible).

Nextcloud advises to use the Ubuntu server 14.04 LTS as the operating system. This can be easily installed using an USB-stick and [this guide][ubuntuguide]. It also explains how to install a LAMP stack. But what is a LAMP stack?

[ubuntuguide]: http://www.tecmint.com/ubuntu-14-04-server-installation-guide-and-lamp-setup/

But what is a LAMP stack of which I am talking about. LAMP stands for Linux Apache MySQL PHP, which defines the setup for the server. The main operating system being Linux (so for instance Ubuntu).

Apache is the webserver, you can see it as the manager of your server. Apache handles the requests by the visitors of your server. So if somebody for instance types in your website address which is hosted by your server than the Apache software tells your server what to do and Apache makes sure the request is handled and 'served' back to that somebody.

MySQL is the database. Here is where all the data is stored. Apache can for instance request data from the database. You can see the database as one big closet filled with all your stuff. MySQL then knows exactly where all your stuff is stored, however you have to correctly ask for it in order to receive it. These questions to the database are called queries.

Finally PHP. This is used for running programs by the webserver. It can manipulate the data given by the database for instance. It also creates the pages that the Apache webserver then can send back to the visitor of your site.

So imagine you are on a forum. The page in the browser that you see is a page that you have requested. So once you pressed enter after typing in the URL-address your web browser sends a request to the server that runs the forum of your URL-address. Now the webserver (for instance Apache) receives this request and handles it and runs the PHP-script creating a html-page. If there is data needed from the database, for instance other posts on the forum, the PHP-script will get this from the database and uses it to create the html-page. Once this html-page is ready the Apache server will send it back to you and your web browser will present the html-page to your screen.

Nextcloud also uses this LAMP setup, however in a much more complex way. Still the basics are the same: Linux is the operating system, Apache handles the requests, MySQL is the database and PHP is the programming language. It is also possible to use a different setup. Instead of Apache there is also Nginx, or instead of MySQL there is also PostGreSQL or instead of PHP, Python is also often used. Now that we know how servers work in general let's continue with the setup of Nextcloud.

Nextcloud installation
------
Well once you've got your Ubuntu server up and running it is time for the actual Nextcloud installation. The [Nextcloud 9 Server Administration Manual][nextcloud-admin-manual] has a quite straight forward procedure described to [install Nextcloud][nextcloud-installation-manual] manually. It is sometimes a little unclear since it sends you to different pages for specific configurations while you don't even know whether you need them. (Or it is probably just me being a noob). Anyway, it states all the packages you'll need to install and then let's you download the Nextcloud server. I will not retype the whole manual, however I will comment on the stuff I found unclear or I had to change for myself to get it to work. For me the 'wget' option did not work since I could not get the PGP signature verified. So I install lynx by,

```terminal
sudo apt-get install lynx
```
[nextcloud-admin-manual]: https://docs.nextcloud.com/server/9/admin_manual/contents.html
[nextcloud-installation-manual]: https://docs.nextcloud.com/server/9/admin_manual/installation/source_installation.html

Lynx is a text-based web browser so you're able to browse the web without a graphical interface. You can read the entire manual but the usage is quite easy, just be careful hitting the arrow-keys as for instance the <- arrow key actually means 'previous page'. In short you'll need the following keys:

* 'G': for Go, now you are able to type a URL-address
* Up- and down: this goes through all the clickable links on the page
* Enter: to 'click' on the selected link
* 'D': to verify that you would like to download a file when clicked on a 'download'-button
* 'Y': to verify that you would like to save the downloaded filled
* 'Q': to quit

The fifth point is actually quite important. When you forget this step the file will not be saved and you'll have to download the file again. I've made this mistake several times already. Just continue the manual from here on.

After downloading the Nextcloud server there is a header stating 'BINLOG_FORMAT = STATEMENT' which in my opinion should not be there but maybe later in this page. It is about an error message you could get during installation. However, skip it for now since we're not installing yet, first the apache web server will need to be configured.

During the Apache configurations it says somewhere the if you're running the 'mod_fcgi' instead of the standard 'mod_php' then you should enable another module as well. It has something to do with the way PHP is run by your web server. You can read about it [here][PHP_mode1] or [here][PHP_mode2] or google it yourself, however I could not find a way to check which of mode I am running. So I'll just assume 'mod_php' and continue.

[PHP_mode1]: http://blog.layershift.com/which-php-mode-apache-vs-cgi-vs-fastcgi/
[PHP_mode2]: https://www.chriswiegman.com/2011/10/fastcgi-vs-suphp-vs-cgi-vs-mod_php-dso/

Once you get to the section 'Installation Wizard' go to the link that says 'Setting Strong Directory Permissions'. There read the page and create the bash script they provide. Make sure that once you've created the script you'll make it executable (seems trivial) and then you can run it. Then go back to the installation manual. Before you continue on the page 'Installation Nextcloud From the Command Line' you first have to create a [database][database-manual] (yes, click it for the instructions).

[database-manual]: https://docs.nextcloud.com/server/9/admin_manual/configuration_database/linux_database_configuration.html#db-binlog-label

When creating the 'mysql.ini' file, note that they give the wrong path (I think). Do not **not** use,

```terminal
/etc/php5/conf.d/mysql.ini
```

But use,

```terminal
/etc/php5/apache2/conf.d/mysql.ini
```

When creating a database make sure you just use the exact names as they propose. Do not try to be smart and change 'username' into something different because apparently that does not work. Even let the password be 'password' *sigh*. It seems that somewhere in the installation procedure those values are hard-coded. So even if you adjust the way you run the 'occ maintenance:install' procedure to follow your custom username, it does not seem to work. I had to delete the first database and created a new one, just typed everything exactly like they told me to. Then I ran

```terminal
sudo -u www-data php /var/www/nextcloud/occ  maintenance:install --database "mysql" --database-name "nextcloud"  --database-user "root" --database-pass "password" --admin-user "admin" --admin-pass "password"
```

(due to the changed permissions I was not allowed in the '/var/www/nextcloud/' directory anymore, but this worked fine) and I got,

```terminal
Nextcloud is not installed - only a limited number of commands are available
ownCloud was successfully installed
```

So it worked! Little shocked it said 'ownCloud' though, but that's probably since it's based largely on ownCloud I guess?

Nextcloud configuration
------

So when I went to my local ip-address of my UP-board and the page of nextcloud it gave an error. You tipe in the browser,

```
http://your-ip-address/nextcloud
```

And it said something about untrusted domain. That is because Nextcloud only has 'localhost' as a trusted domain in it config file. This config file is however owned by the HTTP user (www-data). Adjust the file by sudo'ing nano to this config.php file and add its own local ip-address as a trusted domain. Now it should work and you can log in using 'admin' and 'password' (please change this right away).

Port-forwarding
------
Well now you should look into your router and see if it supports port-forwarding. It means that when you go to your ip-address in a browser somewhere and add that port, your router knows that you want to connect to your Nextcloud server and it makes the connection. My advice is not to use a regular port, that makes it harder for others to find it (brings a little security?). You'll have to change your VirtualHost files too then though, since the web server also has to listen to that specific port. Just go to,

```terminal
sudo nano /etc/apache2/ports.conf
```

and add,

```nano
Listen 'port-of-your-choice'
```

Then go to,

```terminal
sudo nano /etc/apache2/sites-available/000-default.conf
```

and add,

```nano
<VirtualHost *:port-of-your-choice>
  ServerAdmin webmaster@localhost
  DocumentRoot /var/www/nextcloud
  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Save and restart the apache web server,

```terminal
sudo service apache2 restart
```

And it should work! Enjoy!
