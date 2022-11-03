# Overview
The page includes documentation for the portal and admin sites.  This includes information about the following 
applications:
* the UK **portal** flask application (portal.satsense.com);
* the **admin site** flask application (admin.satsense.com);
* the **tileserver** for the full UK portal (portal.satsense.com/tiles);
* the **demo portal** PHP application (demo.satsense.com);
* the **demo tileserver** for the demo portal (demo.satsense.com/tiles).

We will refer to each application by the text in **bold**. Since both the portal and the admin site are written in 
flask, this documentation starts with detailing information common to both.  We then document details for both 
tileservers and for the demo portal. We finish with troubleshooting hints for all applications, which also cover where
the reader can find relevant log files. 

Contents:

* [General Information for Flask Applications](#general-information-for-flask-applications)
    * [Documentation and Resources](#documentation-and-resources)
    * [Project Design Choices](#project-design-choices)
    * [Testing](#testing)
    * [Release Set Up](#release-set-up)
* [Portal](#portal)
    * [Release Portal](#release-portal)
    * [Virtual Host for Portal](#virtual-host-for-portal)
* [Admin Site](#admin-site)
    * [Release Admin Site](#release-admin-site)
    * [Virtual Host for Admin Site](#virtual-host-for-admin-site)
* [Tileserver and Demo Tileserver](#tileserver-and-demo-tileserver)
* [Demo Portal](#demo-portal)
* [Troubleshooting](#troubleshooting)

# General Information for Flask Applications

## Documentation and Resources
The reader may find the following resources useful:
* [Flask Documentation](https://flask.palletsprojects.com/en/1.1.x/) - general documentation for flask.
* [Flask Login Documentation](https://flask-login.readthedocs.io/en/latest/) - documentation for the flask plugin we 
have chosen to handle user sessions.
* [Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world) - an 
introduction to the world of flask, written in 2017-18.  Suitable for absolute beginners to flask, but also useful for 
seasoned developers.  However, bear in mind that Miguel hasn't necessarily chosen the same design choices that we have.
* [Explore Flask](http://exploreflask.com/en/latest/index.html) - a book backed on Kickstarter in July 2013.  Most of 
the material is still relevant.

## Project Design Choices

### Overall Design

From the root of the flask app repository you should find the following directory structure:

```
- <name of project python module>
    - app
    - **possibly some other python modules**
- src
    - js
    - css
    - **possibly some other directories/files**
- requirements
    - development.txt
    - production.txt
- tests
    - integrations_tests
    - **possibly some other tests (e.g. unit tests)**
- app.py
- config.ini
- setup.py
- .gitlab-ci.yml
- MANIFEST.in
- pytest.ini
- **some other files**
```

The directory `<name of project python module>` contains all of the python code relevant to the site. The actual flask 
app is contained in the [subdirectory `app`](#name-of-project-python-moduleapp-design), and the 
`<name of project python module>` directory may contain other directories - for example, a `domain` directory to allow 
the app to connect to a database.

The directory `src` contains javascript and styling code, amongst other files.  This directory contains the versions
of this code that is checked into source control, that is, it is easy for a developer to work on them.  However, before 
the files are able to be served by a server (development or production), this code needs to be compiled and possibly 
bundled.  See specifics in each projects' `readme.md`.

The directories described above are generally all a developer would have to change to introduce new features.  However, 
here is a brief description of the rest:
* The directory `requirements` contains pip dependencies for the projects.  Instructions for how to install dependencies 
should be found in each projects' `readme.md`. 
* The directory `tests` contains tests for the project. See the [testing section](#testing) for information about testing 
structure choices for the projects.  We use `pytest` as the project test runner, so setting are contained in 
`pytest.ini`.
* The file `app.py` contains bootstrap code that runs the project.  Running `flask run` (after setting up the 
enviroment etc) runs this file.
* The file `.gitlab-ci.yml` instructs gitlab to automatically run the tests for the project upon pushing a commit
* The file `setup.py` allows the project python module to be installable into a virtual or conda environment.
* Upon installing the project into an environment, non-python files (such as html) may not be included in to the 
installation. The file `MANIFEST.in` specifies files that should be included into the installation.


### `<name of project python module>/app` Design

We aim to follow an [Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) (MVC) 
design pattern.  With that in mind, the `<name of project python module>/app` directory contains the following 
structure:
```
- controllers
- models
- templates
- **possibly some other files/directories**
```
The directory `templates` contains the `Views` with regards to the MVC pattern - whilst this mismatch of terminology 
may be confusing at first, it is convention for flask to look for a directory called `templates`. This directory
contains [jinja2](https://jinja.palletsprojects.com/en/2.11.x/) and `.txt` templates.  Jinja2 templates are html that 
allow dynamic values to be injected into them.  For example:
```
<p>Hello, {{ model.name }}</p>
```
expects an object to be injected into the page with the attribute `name`.

We call these objects "models".  These are exactly the objects you can find in the `models` directory.  Because models
are extremely simple classes, we aim to use python dataclasses for the models.

Whilst Jinja2 templates expect a model to be injected into them, we still need code that will obtain data (perhaps 
from a database), initialise a model and then actually do the injection of the model into the template.  This code
is known as a controller and it lives inside the `controllers` directory. 
 
We aim to use the same names across the different directories, so it clear to the developer where to find related 
models, templates (views) and controllers. For example, the developer may expect to find something like following 
structure:
```
- controllers
    - manage_users_controller.py
- models
    - manage_users_models.py
- templates
    - manage_users
        - view_user.html
        - search_users.html
```

## Testing

TODO

## Release Set Up

We host the flask applications on the api server `satsense-api.mb1.unipart.io`.  The following describes how the flask 
application accepts requests:

1. User makes http request to api server;
2. [Apache httpd server](https://httpd.apache.org/) accepts request and proxies it to a port (e.g. port 5000) also on 
api server;
3. Flask application listens on port (e.g. port 5000) and accepts proxied request, it then responds appropriately. 
To note, [systemd](https://systemd.io/) runs a [gunicorn server](https://gunicorn.org/) allowing the flask application 
to listen on port.
4. User receives response after it has been proxied back through httpd server.

In the following, we describe how to get the flask application up and running as per step 3.  
See the [portal](#portal) and [admin site](#admin-site) sections for further information and example virtual
host files for apache httpd.  For further information about virtual hosts, see the 
[miscellaneous web section](webmisc.md), where there is also information about setting up DNS records, and ssl 
certificates (which will also need to be done for the application).

### Flask Application Set Up

#### Overview of directory structure

All the code for the applications is stored within a subdirectory of `/var/www`.  For example, the live portal website 
code is contained within:
```
/var/www/flask-portal-live
```
This subdirectory has the following structure:
```
- app_home
- log   
- public 
- virtual_environments
```

The app is installed into a virtual environment inside `virtual_environments`. The directory `app_home` contains 
bootstrap code needed for the application, namely `app.py` and `config.ini` (see [Overall Design](#overall-design)). 

The directory `public` contains static assets that are not served by the flask application, these assets include (but 
not limited to):
* static html (such as error pages);
* javascript;
* css;

Apache is configured to serve these static assets directly rather than proxy the request to flask.

Finally, the `log` directory contains logs for application.  Any errors/requests that the apache server runs into can 
be found in this directory.  The flask application itself is also configured to report any errors, these can be found
in the sub directory `log/flask`. 

#### Commands for setting up directory structure

In the following, we assume the location of the live portal.  The reader should change the folder paths for the 
application they are setting up.

If this is the first application on the server, it is best to add a user to handle the flask applications:
```
useradd flaskuser
```

Now we make the directories as per the [overiew of directory structure](#overview-of-directory-structure) section:

```
sudo mkdir /var/www/flask-portal-live/
cd /var/www/flask-portal-live/

sudo mkdir public/
sudo chown flaskuser:apache public/

sudo mkdir log/
sudo chown flaskuser:apache log/

sudo mkdir log/flask
sudo chown flaskuser:flaskuser log/flask/

sudo mkdir app_home/
sudo chown flaskuser:flaskuser app_home/

sudo mkdir virtual_environments/
sudo chown flaskuser:flaskuser virtual_environments/
```

Notice that `flaskuser` is the owner of all of the directories, but that apache needs access to the `public` and `logs`
directories.  The former is so that the the apache server can serve files from this directory without having to give
send a proxy request to the flask application. The latter is so the apache server can write logs to this directory. To
write logs, we have to also give that folder SELinux permissions to do that: 

```
sudo semanage fcontext -a -t httpd_log_t "/var/www/flask-portal-live/log(/.*)?"
sudo restorecon -R -v log/
```

You can now populate the virtual environment, and app_home/public directories.  Further details can be found in the
[Release Portal](#release-portal) and [Release Admin Site](#release-admin-site)

#### systemd and gunicorn set up 

The following assumes the setup for the live portal web site and that the application has been set up as per the 
[Release Portal](#release-portal) section.  Replace names/paths that make sense for the application you would like to 
run via systemd.

Gunicorn is easy to install via pip. First make sure you are logged into the api server as `flaskuser` (you can run
`sudo su flaskuser` as `centos`), and then run:
```
source /var/www/flask-portal-live/virtual_environments/venv/bin/activate
pip install gunicorn
```

If you wish, you can check that the site is set up correctly (i.e. that the application will run) by running:

```
gunicorn -b localhost:<port> -w 1 app:app
```
from `/var/www/flask-portal-live/app_home` and replace `<port>` with a port number.

We'd like the above process to be run automatically, which is done by using `systemd`. To set up `systemd`, you'll
need sudo privileges (which `flaskuser` does not have).  First, create a file:
```
touch /etc/systemd/system/portallive.service
chmod 664 /etc/systemd/system/portallive.service
```

Then populate the file, the following is the current `systemd` config for the live portal process:
```
[Unit]
Description = PortalLive
After = network.target

[Service]
User=flaskuser
WorkingDirectory=/var/www/flask-portal-live/app_home
ExecStart=/var/www/flask-portal-live/virtual_environments/venv/bin/gunicorn -b localhost:7000 -w 4 app:app
Restart=always

[Install]
WantedBy = multi-user.target
```

For systemd to recognise the new file, you will need to run
```
systemctl daemon-reload
```
and you can check it has been found using 
```
systemctl list-unit-files
```

The service can be started via:
```
systemctl start portallive
```

or stopped via:
```
systemctl stop portallive
```

To allow the service to start on server reboot, it needs to be enabled:
```
systemctl enable portallive
```

# Portal

## First Release of Portal

At time of writing, the user "flaskuser" on the api server runs all of the flask apps.  The following instructions 
assume that the portal has already been released on the server. 

---
**If this is the first release of the portal on the server**

The reader can follow the instructions following this note, but omit uninstalling satdom/satportal and removing static files 
(just copy them over).  In addition to the instructions following this note, the will reader will need to:

* Create a virtual environment for the application
* Populate the app_home directory
* Create a 503 document in public/error


the reader will also need to copy the app.py and config.ini files 
(and populate the config.ini correctly):
```
cp app.py /var/www/flask-portal-live/app_home
cp config.ini /var/www/flask-portal-live/app_home
# populate config.ini correctly
```

Finally, gdal is a fiddly dependency and needs to be installed in a particular way.  Instead of just running
```
pip install -r requirements/production.txt
```
check the project readme.md.

##  Subsequent Releases of Portal


---

To release the portal, follow the steps:

1.  Log onto api server as flaskuser (can run `sudo su flaskuser` as centos).
2.  Checkout latest versions of the master branches of satsense-domain, and satsense-portal. Projects are already cloned 
in `~/projects`.
3.  Change dir to satsense-portal root directory (i.e. `cd ~/projects/satsense-portal`).  Make sure 
    1. the `./static` folder doesn't exist or is empty;
    2. that `src/js/config.js` is correct and that the rest of the working directory is clean.  Backups can be found 
    at `~/config_backups`.
4.  From satsense-portal root directory, run `npm run gulp`.

The following will actually interact with the current running site.  It is advisable to bring the site down whilst doing 
these steps.  This can be done via `sudo systemctl stop portalstaging.service` or 
`sudo systemctl stop portallive.service` for the live version.

5. Apply any db changes using alembic. (This step is a stub, more details is a TODO)

6. Check if the config needs updating.  On gitlab, 
   [compare](https://gitlab.com/SatSenseLtd/satsense-portal/-/compare?from=master&to=master) the master branch of 
   satsense-portal to the most recent release tag - if `config.ini` has been updated, you'll need to apply changes to 
   the  current version. Current site config can be found at `/var/www/flask-portal-live/app_home/config.ini`.  Make 
   a back up if you wish.

7. Uninstall satdom/satportal in the venv:
    1. `source /var/www/flask-portal-live/virtual_environments/venv/bin/activate`
    2. `pip uninstall satdom satportal`
    
8. Check dependencies, on the same gitlab comparison as config update, check if `requirements/production.txt` has been 
updated.  If so, run (within venv):
    1. `pip install -r requirements/production.txt`

9. Reinstall satdom/satsense-portal into the venv. From within venv, run:
    3. `cd ~/projects/satsense-domain`
    4. `pip install --no-dependencies .`
    5. `cd ~/projects/satsense-portal`
    6. `pip install --no-dependencies .`

10. Replace static files
    1. `cd /var/www/flask-portal-live/public/static`
    2. `rm -rv *`
    3. `cp -rv ~/projects/satsense-portal/static/* /var/www/flask-portal-live/public/static`

Run `sudo systemctl start portallive.service` and check the site works as expected.

## Virtual Host for Portal

For further information about virtual hosts, see the [apache httpd set up](webmisc.md#apache-httpd-set-up).

Here is the current virtual host for the live portal:
```
<VirtualHost *:80>

ServerName portal.satsense.com
ServerAlias portal.satsense.com
DocumentRoot /var/www/flask-portal-live/public

# rewrite to https
RewriteEngine on
RewriteCond %{SERVER_NAME} =portal.satsense.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

</VirtualHost>
<VirtualHost *:443>

ServerName portal.satsense.com
ServerAlias portal.satsense.com
DocumentRoot /var/www/flask-portal-live/public

# flask proxy
ProxyPass /static !
ProxyPass /error !
ProxyPass /tiles !
ProxyPass / http://localhost:5001/
ProxyPassReverse / http://localhost:5001/

# logs
ErrorLog /var/www/flask-portal-live/log/error.log
CustomLog /var/www/flask-portal-live/log/requests.log combined

# headers:
Header always set Strict-Transport-Security "max-age=31536000; includeSubdomains"
Header always set X-Frame-Options "deny"
Header always set X-Xss-Protection "1; mode=block"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "no-referrer"
Header always edit Set-Cookie ^(.*)$ $1;HttpOnly;Secure;SameSite=Strict

# Error Documents
ErrorDocument 503 /error/503.html

# wsgi
WSGIScriptAlias /tiles /var/www/tileserver-live/Tileserver/tileserver.wsgi
WSGIDaemonProcess tileserver_live python-home=/var/www/venvs/tileserver-live processes=4 threads=1
WSGIProcessGroup tileserver_live

# ssl
SSLCertificateFile /etc/letsencrypt/live/portal.satsense.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/portal.satsense.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/portal.satsense.com/chain.pem

</VirtualHost>
```

In the above:

*   The httpd server listens to `http` requests on port `80`; the configuration inside 
    `<VirtualHost *:80>` is used upon a `http` request to `portal.satsense.com`.
    The only instruction is to rewrite the request to a `https` request.
*   The httpd server listens to `https` requests on port `443`; the configuration inside 
    `<VirtualHost *:443>` is used upon a `https` request to `portal.satsense.com`.
*   `DocumentRoot` specifies the directory where the server should look for files.  Without any other 
    configuration, a request to `https://portal.satsense.com/index.html` would server the file `index.html` 
    inside the `DocumentRoot` directory.
*   The `flask proxy` session allows the httpd server to forward requests to port `5001` (where the flask 
    application is listening).  Values with an exclamation mark (e.g. `ProxyPass /static !`) mean that requests
    following this pattern should not be proxied (e.g. `https://portal.satsense.com/static/resource.js`)
*   The `logs` section specifies where the httpd logs should be made.
*   The `headers` section sets headers on each response, these values are recommended to improve the security 
    between the server and the client
*   The `Error Documents` are documents that are served upon a particular error thrown by the server. A `503`
    corresponds to a `Service Unavailable` and is thrown if the flask server is not running - therefore any
    website users see a branded error page and not the default httpd error page.
*   The `wsgi` section sets up the tileserver.  Requests to `https://portal.satsense.com/tiles` are forwarded
    to the tileserver. See the section about [installing the tileserver](#release-a-new-tileserver).
*   The `ssl` section tells the httpd server which ssl certificate to use. **This section is automatically 
    populated by [certbot](webmisc.md#ssl-certificates) and should not be populated manually upon a new 
    release**.

# Admin Site

## Release Admin Site

At time of writing, the user "flaskuser" on the api server runs all of the flask apps.  The following instructions 
assume that the admin site has already been released on the server. 

---
**If this is the first release of the admin site on the server**

The reader can follow the following instructions, but omit uninstalling satdom/satadmin.  The reader will also 
need to copy the app.py and config.ini files (and populate the config.ini correctly):
```
cp app.py /var/www/admin-live/app_home
cp config.ini /var/www/admin-live/app_home
# populate config.ini correctly
```

The reader will also need to prepare the 503 error files. (more details is a TODO)

---

To release the admin site, follow the steps:

1.  Log onto api server as flaskuser (can run `sudo su flaskuser` as centos).
2.  Checkout latest versions of the master branches of satsense-domain, and satsense-admin. Projects are already cloned 
in `~/projects`.
3.  Make sure both projects are clean checkouts.
4.  From satsense-admin root directory, run `npm run-script build`.

The following will actually interact with the current running site.  It is advisable to bring the site down whilst doing 
these steps.  This can be done via `sudo systemctl stop adminstaging.service` or 
`sudo systemctl stop adminlive.service` for the live version.

5. Apply any db changes using alembic. (This step is a stub, more details is a TODO)

6. Check if the config needs updating.  On gitlab, 
   [compare](https://gitlab.com/SatSenseLtd/satsense-admin/-/compare?from=master&to=master) the master branch of 
   satsense-admin to the most recent release tag - if `config.ini` has been updated, you'll need to apply changes to 
   the  current version. Current site config can be found at `/var/www/admin-live/app_home/config.ini`.  Make 
   a back up if you wish.

7. Uninstall satdom/satadmin in the venv:
    1. `source /var/www/admin-live/virtual_environments/venv/bin/activate`
    2. `pip uninstall satdom satadmin`
    
8. Check dependencies, on the same gitlab comparison as config update, check if `requirements/production.txt` has been 
updated.  If so, run (within venv):
    1. `pip install -r requirements/production.txt`

9. Reinstall satdom/satadmin into the venv. From within venv, run:
    3. `cd ~/projects/satsense-domain`
    4. `pip install --no-dependencies .`
    5. `cd ~/projects/satsense-admin`
    6. `pip install --no-dependencies .`

Run `sudo systemctl start adminlive.service` and check the site works as expected.

## Virtual Host for Admin Site

For further information about virtual hosts, see the [apache httpd set up](webmisc.md#apache-httpd-set-up).

Here is the current virtual host for the live admin site:
```
<VirtualHost *:80>

ServerName admin.satsense.com
ServerAlias admin.satsense.com
DocumentRoot /var/www/admin-live/public

RewriteEngine on
RewriteCond %{SERVER_NAME} =admin.satsense.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

</VirtualHost>
<VirtualHost *:443>

ServerName admin.satsense.com
ServerAlias admin.satsense.com
DocumentRoot /var/www/admin-live/public

# flask proxy
ProxyPass /error !
ProxyPass / http://localhost:6001/
ProxyPassReverse / http://localhost:6001/

# logs
ErrorLog /var/www/admin-live/log/error.log
CustomLog /var/www/admin-live/log/requests.log combined

# headers:
Header always set Strict-Transport-Security "max-age=300; includeSubdomains"
Header always set X-Frame-Options "deny"
Header always set X-Xss-Protection "1; mode=block"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "no-referrer"
Header always edit Set-Cookie ^(.*)$ $1;HttpOnly;Secure;SameSite=Strict

# Error Documents
ErrorDocument 503 /error/503.html

SSLCertificateFile /etc/letsencrypt/live/admin.satsense.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/admin.satsense.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/admin.satsense.com/chain.pem
</VirtualHost>
```

In the above:

*   The httpd server listens to `http` requests on port `80`; the configuration inside 
    `<VirtualHost *:80>` is used upon a `http` request to `admin.satsense.com`.
    The only instruction is to rewrite the request to a `https` request.
*   The httpd server listens to `https` requests on port `443`; the configuration inside 
    `<VirtualHost *:443>` is used upon a `https` request to `admin.satsense.com`.
*   `DocumentRoot` specifies the directory where the server should look for files.  Without any other 
    configuration, a request to `https://admin.satsense.com/index.html` would server the file `index.html` 
    inside the `DocumentRoot` directory.
*   The `flask proxy` session allows the httpd server to forward requests to port `6001` (where the flask 
    application is listening).  Values with an exclamation mark (e.g. `ProxyPass /error !`) mean that requests
    following this pattern should not be proxied (e.g. `https://admin.satsense.com/static/resource.js`)
*   The `logs` section specifies where the httpd logs should be made.
*   The `headers` section sets headers on each response, these values are recommended to improve the security 
    between the server and the client
*   The `Error Documents` are documents that are served upon a particular error thrown by the server. A `503`
    corresponds to a `Service Unavailable` and is thrown if the flask server is not running - therefore any
    website users see a branded error page and not the default httpd error page.
*   The `ssl` section tells the httpd server which ssl certificate to use. **This section is automatically 
    populated by [certbot](webmisc.md#ssl-certificates) and should not be populated manually upon a new 
    release**.



# Demo Portal

TODO

# Troubleshooting

Use the following tables to look up the actions you should do. Actions are listed in the order you should consider them. 
Instructions for each action are listed below the tables.

### Troubleshooting for Flask Applications (Portal and Admin Site)

| Problem  |  Actions  |
|----------|----------- |
| Web site isn't responding. For example `https://portal.satsense.com` or `https://admin.satsense.com` don't respond. |  <ul><li>Check Api server is running</li><li>Check httpd errors</li><li>Check a record</li></ul> |
| Web site reports "Unexpected error occurred" |    <ul><li>Check flask errors</li></ul> |
| Web site reports "Server Unavailable" |    <ul><li>Check gunicorn systemd process is running</li></ul> |
| Web site responds, but logging in does not work | <ul><li>Check flask errors</li><li>Restart flask application</li></ul> |

### Troubleshooting for Demo Portal

| Problem  |  Actions  |
|----------|----------- |
| Web site isn't responding. | <ul><li>Check Api server is running</li><li>Check httpd errors</li><li>Check a record</li></ul> |
| Web site reports "Unexpected error occurred"  |  <ul><li>Check httpd errors</li><li>Httpd restart</li></ul> |

### Actions 

* **Check Api server is running.** Log into api server using 
    ```
    ssh -p 2222 centos@satsense-api.mb1.unipart.io
    ```
    If no response, contact unipart sysadmins.

* **Check httpd errors.** Log files for httpd are kept:
    ```
    Portal + Tileserver:
    /var/www/flask-portal-live/log/error.log
    /var/www/flask-portal-live/log/requests.log
   
    Admin site:
    /var/www/admin-live/log/error.log
    /var/www/admin-live/log/requests.log
  
    Demo Portal + Demo Tileserver:
    /var/www/demo-portal-live/log/error.log
    /var/www/demo-portal-live/log/requests.log
    ```
* **Httpd restart.** Restart the httpd process via:
    ```
    sudo systemctl restart httpd.service
    ```
* **Check flask errors.** Log files for flask errors are kept in the following directories:
    ```
    Portal:
    /var/www/flask-portal-live/log/flask/
   
    Admin site:
    /var/www/admin-live/log/flask
    ```
* **Check gunicorn systemd process is running.** Log into api server, and run one of:
    ```
    systemctl status portallive
    systemctl status adminlive
    ```
    If process is reported as stopped. Then run
    ```
    sudo systemctl start portallive
    sudo systemctl start adminlive
    ```
    Check flask errors for anything untoward, especially if starting the process doesn't work.
    
*  **Restart flask application.** Stop and start the application process.  For the portal:
    ```
    sudo systemctl stop portallive
    sudo systemctl start portallive
    ```
    and similarly for the admin site.
    
*  **Check a record.** This is very unlikely to be the problem.  Only do this step if the api server is not receiving
    requests.  For example, there should be logs in `/var/www/flask-portal-live/log/requests.log`.  
    1.  Log in `www.namebright.com` and go to `DNS Records`. 
    2.  Check details are correct next to problematic subdomain. 
    
*  **Check tileserver config vs overview geotiffs.** Open tileserver config:
   ```
   # for tileserver
   vi -R /var/www/tileserver-live/Tileserver/config.ini
   # for demo tileserver
   vi -R /var/www/tileserver-demo/Tileserver/config.ini
   ```
   You should see a json object for the value for `config` under the section `[tilestache]`.  Inside this object,
   there should be one or more `layers`, each of which have values for `geotiff_directory` and `geotiff_name_stem`.
   Check:
   * The `geotiff_directory` exists;
   * Files with names `<geotiff_name_stem>_<resolution>.geotiff` exist for each resolution in `resolutions` under the 
   section `[geotiff]`.
   * None of these files are corrupt.    
   