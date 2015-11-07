#Configure Apache on Debian Clients

##Prerequisites

Test that MySQL is running

```bash
mysqladmin -u root -p status
```

Test that Apache is running

```bash
sudo apachectl -t
```

##Step 1: Set DocumentRoot and ServerName

```bash
sudo vim /etc/apache2/apache2.conf
```

##Step 2: Create a VirtualHost file for a site

```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/<your_site>.conf

# open the new .conf file
vsudo vim /etc/apache2/sites-available/<your_site>.conf

# set it to point to your site
<VirtualHost *:80>
  ServerAdmin you@domain.com
  DocumentRoot /path/to/site/root
  ServerName your.site
  ServerAlias your.site
  Errorlog ${APACHE_LOG_DIR}/error.log
  Customlog ${APACHE_LOG_DIR}/access.log
</Virtualhost>
```

##Step 3: Enable Site

```bash
sudo a2ensite <your_site>
```

##Step 4: Add record to host machine's /etc/hosts file

```bash
<guest ip> <your.site>
```
If OS X, flush DNS Cache

```bash
dscacheutil -flushcache
```


