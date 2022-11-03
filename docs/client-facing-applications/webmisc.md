# Overview

Contents:

* [Apache Httpd Set Up](#ssl-certificates)
* [SSL Certificates](#ssl-certificates)
* ['A' Records](#a-records)
* [Application Emails](#application-emails)

### Apache Httpd Set Up

We follow the `sites-available` pattern for setting up our virtual hosts.  This involves creating folders:

```
sudo mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled
```

and then populating them with virtual host files.  We manually put in these files inside `/etc/httpd/sites-available`
and then we create and destroy symbolic links to those files inside `/etc/httpd/sites-enabled`.  

The apache server is only aware of the `/etc/httpd/sites-enabled` folder. To do this, open the main config file:

```
sudo vi /etc/httpd/conf/httpd.conf
```

and add in
```
IncludeOptional sites-enabled/*.conf
```

somewhere in the file.

We can now create a virtual host config file
```
sudo vi /etc/httpd/sites-available/portal-live.conf
```
and populate it. Example virtual host files are contained in the documentation for each application.

Now to make the site enabled, create a symbolic link:
```
sudo ln -s /etc/httpd/sites-available/portal-live.conf /etc/httpd/sites-enabled/portal-live.conf
```

The site can be taken offline by removing this symbolic link. After adding or removing a symbolic link, the httpd 
process will need to be restarted:
```
systemctl restart httpd.service
```

### SSL Certificates

We use [certbot](https://certbot.eff.org/) to produce our ssl certificates.  There are very good instructions on that 
website, see [here](https://certbot.eff.org/lets-encrypt/centosrhel7-apache) for installation instructions for Centos 7 
using apache.  Note that centos has EPEL enabled by default, but you can check whether this is the case via:
```
rpm -q epel-release --last
```

Running step five (`certbot --apache`) will ask for:
* An email for urgent renewals/security notices.
* Ask for you to agree to the terms of service.
* Ask if you'd like to share email with EFF.
* Ask which domains you'd like to activate https for (you can select domains that already have a certificate if you want to add a domain to an existing certificate).
* Ask if you'd like to redirect to https.

Note that if certbot runs into a problem, it will attempt to rollback any config to the before it was run.  If config installation was unsuccessful, running `certbot --apache` should recognise if any certificate has been obtained previously, and ask if you'd like to use that.

Running 
```
echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew" | sudo tee -a /etc/crontab > /dev/null
```
adds a chron job every 12 hours (offset by a random amount) and sets the cert to be renewed.  The certificate is only 
renewed if certificates are short lived.

**Troubleshooting**

On first run through, got the error `Name duplicates previous WSGI daemon configuration`. See 
[this stack overflow post](https://stackoverflow.com/questions/47803081/certbot-apache-error-name-duplicates-previous-wsgi-daemon-definition).

### 'A' Records 

Our domain is registered with `www.namebright.com`.  To route a new subdomain, for example 
`notaliveexample.satsense.com`
1.  Log in (credentials are deliberately omitted) and go to `DNS Records`. 
2.  Scroll, to the bottom and click `Add a New DNS Record`. 
3.  Select `A Address Record` from the *Choose record type* drop down.
4.  Select `Custom Host Name...` from the *Subdomain* drop down.
5.  Type in `notaliveexample` (or similar).
6.  Type in the IP Address of the server.
7.  Click `Add`.

### Application Emails

Currently the portal and the admin site send emails for logging and account management purposes.  These 
applications use the flask extension [flask-mail](https://pythonhosted.org/flask-mail/). Whilst we assume the reader
is setting up email sending with these applications (and in particular flask-mail), these instructions should be
transferable to any other framework.

To get these applications to send an email, you will need to populate the following settings in `config.ini`:

```
mail_server = smtp.office365.com
mail_port = 587
mail_use_tls = true
mail_use_ssl = false
mail_username = <username>
mail_password = <password>
```

The above settings are correct apart from the username and password.  The username is the email address associated with
your account, but note that the password is **not** the password you use to log in online.  Instead, you will need to 
[set up an application password](https://docs.microsoft.com/en-gb/azure/active-directory/user-help/multi-factor-authentication-end-user-app-passwords).

Additionally, `config.ini` asks for a `default_mail_sender`.  This could be your email address, but you can also specify 
`noreply@satsense.com` to send emails from that anonymous user (this is the current live setting).
 

------
**The current live applications uses credentials for `michael.west@satsense.com`.  As a result, if this account is ever
removed then the emails will stop working on these applications.**
------
