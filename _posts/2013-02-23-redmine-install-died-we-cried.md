---
title: "Our Redmine install died, We all cried!"
date: "2013-02-23"
categories: 
  - "devops-sysadmin"
tags: 
  - "12-10"
  - "2-2"
  - "install"
  - "redmine"
  - "ruby"
  - "ubuntu"
---

We have been using redmine for quite a long time and a few months ago attempted to upgrade from 1.3 to 2.something. Unfortunately I (quite typically) borked the installation and since then its been hobbling along after my attempts to fix it left it crippled.

Yesterday it finally gave up the fight and my attempts to resurrect the installation were futile. After a quick funeral (the eulogy was very touching), and wake in a nearby emporium of alcoholic beverages to commiserate our loss, I set about trying to figure out what to do next. <!--more-->

## Alternatives

Now while Redmine is a worthy tool and has always managed to do what I needed in the past, recently its just not cut the mustard. I've kept toying with the idea of creating our own project management system but as with all in-house projects that we dream up its just never going to happen.

A quick google around our options are to either go for a hosted solution (not possible as we have some very specific requirements regarding our SCM that mean we have to host our own repos for client work) or Redmine (or chilli project).

Yes we looked at a number of other management tools and of them all Redmine is still the closes to what we needed.

## Installation

So I spin up a new server instance of ubuntu 12.10 on the cloud and get to work installing the latest version.

As root I then run through these steps (you should assume that ALL of these steps require you to be root and files should be owned by root)

```
# update/upgrade base installation of ubuntu packages
apt-get update && apt-get upgrade

# install the requisite scm tools that we use
apt-get install git-core subversion mercurial cvs

# set up ruby
apt-get install ruby rubygems libruby ruby-dev

# set up apache & mysql
apt-get install apache2 libapache2-mod-passenger mysql-server mysql-client libmysqlclient-dev

# install imagemagick and the magick wand
apt-get install  imagemagick libmagickcore-dev libmagickwand5 libmagickwand-dev

# create our user and database in mysql 
# replace uniquePassword with your own password
mysql -u root -p -e "create user 'redmine'@'localhost' identified by 'uniquePassword'"
mysql -u root -p -e "create database redmine"
mysql -u root -p -e "grant all on redmine.* to 'redmine'@'localhost'"
mysql -u root -p -e "flush privileges"

# clone redmine code to target location
cd /usr/local/share
git clone git://github.com/redmine/redmine.git

# set apache as the owner of redmine
chown -R www-data:www-data redmine

# move into our new redmine folder
cd redmine

# set up your database configuration
cp config/database.yml.example config/database.yml
vim config/database.yml
```

```
production:
  adapter: mysql2
  database: redmine
  host: localhost
  username: redmine
  password: uniquePassword
```

```
# install bundler gem
gem install bundler

# use bundler to set up redmine installation and without specified dependencies
bundle install --without development test postgresql sqlite

# set up our secret token
rake generate_secret_token

# set up our database and load default configuration
RAILS_ENV=production rake db:migrate
RAILS_ENV=production rake redmine:load_default_data
```

```
# edit /etc/apache2/sites-available/default
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        ServerName mysite.co.uk
        ServerAlias www.mysite.co.uk
        DocumentRoot /usr/local/share/redmine/public
        <Directory />
                Options FollowSymLinks
                AllowOverride None
        </Directory>
        <Directory /usr/local/share/redmine/public>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                allow from all
        </Directory>

        ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
        <Directory "/usr/lib/cgi-bin">
                AllowOverride None
                Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
                Order allow,deny
                Allow from all
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log

        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn

        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```
# restart apache 
service apache2 restart
```

That should be enough for you to have a working installation of redmine ready for you to use/customise

## Additional Config

We typically have additional steps that we would configure for our own installation.

```
# add plugin assets folder
mkdir /usr/local/share/redmine/public/plugin_assets
chown www-data:www-data /usr/local/share/redmine/public/plugin_assets

# enable some additional apache modules
a2enmod rewrite

# disable mod ssl
a2dismod ssl

# install gnutls 
apt-get install libapache2-mod-gnutls

# install ssl certificate bundle and key (this assumes that you have already copied the key and bundle to ~/)
mv ~/my_certificate.bnd /etc/ssl/certs/my_certificate.bnd
chmod 0644 /etc/ssl/certs/my_certificate.bnd
mv ~/my_certificate.crt /etc/ssl/private/my_certificate.key
chmod 0600 /etc/ssl/private/my_certificate.key
```

```
# now configure your /etc/apache2/sites-available/default-tls
<IfModule mod_gnutls.c>
<VirtualHost _default_:443>
    ServerAdmin webmaster@localhost
    ServerName mysite.co.uk
    ServerAlias www.mysite.co.uk
    DocumentRoot /usr/local/share/redmine/public
    <Directory />
        Options FollowSymLinks
        AllowOverride None
    </Directory>
    <Directory /usr/local/share/redmine/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    # Possible values include: debug, info, notice, warn, error, crit, alert, emerg.
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/ssl_access.log combined

    GnuTLSEnable On
    GnuTLSCertificateFile /etc/ssl/certs/my_certificate.bnd
    GnuTLSKeyFile /etc/ssl/private/my_certificate.key
    GnuTLSPriorities NORMAL:!DHE-RSA:!DHE-DSS:!AES-256-CBC:%COMPAT
</VirtualHost>
</IfModule>
```

```
# Add some Rails / Passenger specific config to /etc/apache2/sites-available/default-tls
RailsEnv production
PassengerDefaultUser www-data
PassengerSpawnMethod smart
PassengerPoolIdleTime 300
PassengerMaxRequests 5000
PassengerStatThrottleRate 5
PassengerHighPerformance On
```

```
# change your /etc/apache2/sites-available/default to redirect to ssl
<VirtualHost *:80>
    ServerAdmin sysadmin@zucchi.co.uk
    ServerName mysite.co.uk
    ServerAlias www.mysite.co.uk
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    Options FollowSymLinks
    AllowOverride None

    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```
# enable your new default-tls vhost and restart apache
a2ensite default-tls
service apache2 restart

# setup &amp; configure email
# when prompted select "internet site" and enter the domain you are hosting redmine from i.e. mysite.co.uk)
apt-get install postfix

# create config file and uncomment the production settings for sendmail
cp /usr/local/share/redmine/config/configuration.yml.example /usr/local/share/redmine/config/configuration.yml
vim /usr/local/share/redmine/config/configuration.yml
```

```
production:
  email_delivery:
    delivery_method: :sendmail
```

```
service apache2 restart
```

```
#install pixel cookers theme cos we like it
git clone git://github.com/pixel-cookers/RedmineThemePixelCookers.git /usr/local/share/redmine/public/themes/pixel-cookers
```
