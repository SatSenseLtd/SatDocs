# Portal/SatShop (satportal)

This is a web GIS application consisting of two major features:

* allows a user to browse SatSense data that is streamed from the server;
* allows a user to order and download SatSense data extracted from the server.

Historically, we have named the project the "portal", but with the addition of the ordering feature we rebranded the site to "satshop".  This documentation may refer to the project by either name. 

We also have a demo portal, but because that is still using a legacy php version of the code, we separate documentation for the demo portal to [here](demo-portal.md).

Instructions for how to install an instance of the portal/satshop into a virtual environment are kept in the project [README](https://gitlab.com/SatSenseLtd/satsense-portal/-/blob/master/README.md).  What that README does not make explicit is where everything is installed on the live server.  We detail that in this documentation, along with how to set up apache to forward on requests.

# Contents

* [Live Server Locations](#live-server-locations) - where files related to the portal are kept on the live server.
* [First Release](#first-release) - how to release a brand new portal.
* [Release Update](#release-update) - how to release an update to a portal that's already been released.
* [Troubleshooting](#troubleshooting) - something not working? No problem.

# Live Server Locations

On the live server, we run several portals, this is so that we can test new datasets/provide other data sets/test new versions without affecting the live site.  This is a list of the portals that are running, along with locations of related important files:

| Portal instance name | Domain end point |  Project directory |  Httpd config file | Systemd file |
|---------------|-------------|----------|----------| -------- |
| satshop | satshop.satsense.com | /var/www/satshop | /etc/httpd/sites-available/satshop.conf | /etc/systemd/system/satshop.service |
| staging | portal.staging.satsense.com | /var/www/flask-portal-staging |  /etc/httpd/sites-available/flask-portal-staging.conf | /etc/systemd/system/portalstaging.service |
| portal2 | portal2.satsense.com | /var/www/flask-portal2 | /etc/httpd/sites-available/flask-portal2.conf | /etc/systemd/system/portal2.service |
| portal3 | portal3.satsense.com | /var/www/portal3 | /etc/httpd/sites-available/portal3.conf | /etc/systemd/system/portal3.service |

The httpd config files are shared with the tileserver httpd files because they are provided by the same domain. 

## Project directory structure 

The structure of each "Project directory" is as follows:

```
- app_home
- log
    - flask   
- public 
    - error
    - static
- virtual_environments
```

The app is installed into a virtual environment inside `virtual_environments`. The directory `app_home` contains bootstrap code needed for the application, namely `app.py` and `config.ini` (see [project root](https://gitlab.com/SatSenseLtd/satsense-portal)). 

The directory `public` contains static assets that are not served by the flask application - the apache server serves them directly. The directory `public/error` houses static error files; when releasing, it is common to bring the flask server down, at which point apache will return the `503` error document in this directory.  The `public/static` folder provides static assets needed for the website (e.g. js, css, images etc), these assets are build from the [project src folder](https://gitlab.com/SatSenseLtd/satsense-portal/-/tree/master/src) during a gitlab build.

The `log` directory contains logs for the application.  Any errors/requests that the apache server runs into can be found in this directory.  The flask application itself is also configured to report any errors, these can be found in the subdirectory `log/flask`. 

# First Release

These are instructions if you need to release a new portal.  The reader will also need to set up any ssl certificates and `A` records. See the [web miscellaneous docs](webmisc.md) for more detail.  If you need to release an update, please see the [Release Update](#release-update) section.  We comment that you may also want to [release a new tileserver](./tileservers.md#first-release) as well.  We assume user/InSAR databases are set up already.

## Set up project directory

In the following, we use a location of a "new-portal".  The reader should change the folder paths for the application they are setting up.

If this is the first flask application on the server, it is best to add a user to handle the flask applications:
```
useradd flaskuser
```

Now we make the directories as per the [Project directory structure](#project-directory-structure) section:

```
sudo mkdir /var/www/new-portal/
cd /var/www/new-portal/

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

Notice that `flaskuser` is the owner of all the directories, but that apache needs access to the `public` and `logs` directories.  The former is so that the apache server can serve files from this directory without having to give send a proxy request to the flask application. The latter is so the apache server can write logs to this directory. To write logs, we have to also give that folder SELinux permissions to do that: 

```
sudo semanage fcontext -a -t httpd_log_t "/var/www/new-portal/log(/.*)?"
sudo restorecon -R -v log/
```

The directories are now set up ready to be populated.

## Populate project directories

We will need to:

1. Download artifacts from the build stage of a [satportal gitlab pipeline](https://gitlab.com/SatSenseLtd/satsense-portal/-/pipelines).
2. Populate the `app_home` directory.
3. Create a 503 document in `public/error`.
4. Populate `public/static` with the static assets.
5. Create a virtual environment for the application and install the project.

We run through each of these steps in succession.  Unless otherwise stated, we will assume that the following statements will be run as `flaskuser`.

### 1. Download satportal artifacts

We keep a shell script on the live server that downloads the artifacts automatically for us.  This is kept in the file:

```
/home/flaskuser/releases/portal/download_portal_artifacts
```

We keep a copy here as well:

```shell
#!/bin/bash

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
CURL_OUTFILE="satportal_artifacts_$TIMESTAMP.zip"
SATPORTAL_BRANCH="master"
HTTP_CODE=$(curl --write-out "%{http_code}" --location --output $CURL_OUTFILE --header "PRIVATE-TOKEN: <gitlab ro access token>" "https://gitlab.com/api/v4/projects/16331419/jobs/artifacts/$SATPORTAL_BRANCH/download?job=create_artifacts")
NEW_DIR="satportal_$TIMESTAMP"
mkdir $NEW_DIR
unzip $CURL_OUTFILE -d $NEW_DIR
rm $CURL_OUTFILE
printf "Response code: $HTTP_CODE\nOut Dir: $NEW_DIR\n"
```

Note that if you use the code above, you should create a [gitlab access token](https://gitlab.com/-/profile/personal_access_tokens) (preferably read only) and replace `<gitlab ro access token>` with 
it. However, the file on the server has an access token pre-filled.

Further, if you need to release a branch other than `master`, then you can change the `SATPORTAL_BRANCH` variable.  For example, at the time of writing (but this is always subject to change), we release a separate branch to "portal3" (whose branch is also called `portal3`).

By convention, we run the `download_portal_artifacts` file from inside its directory, i.e. 

```commandline
cd /home/flaskuser/releases/portal
./download_portal_artifacts
```

and this will create a timestamped directory of the needed artifacts for a satportal release.

### 2. Populate app_home

We assume that the current directory is the timestamped directory created in [the first section](#1-download-satportal-artifacts)

```commandline
cp dist/satportal/app.py /var/www/new-portal/app_home
cp dist/satportal/config.ini /var/www/new-portal/app_home
```

Here, `app.py` is the entry point for the application and `config.ini` is a config file. The latter needs to be edited with config settings for the application (e.g. database strings, live api keys etc).


### 3. Create a 503 document

Create the required error folder:

```commandline
mkdir /var/www/new-portal/public/error
```

And copy the 503 document from the portal artifacts (we assume the current directory is the directory created in [the first section](#1-download-satportal-artifacts)):

```commandline
cp dist/satportal/error/* /var/www/new-portal/public/error
```

### 4. Populate `public/static`

Create the required static folder:

```commandline
mkdir /var/www/new-portal/public/static
```

And copy the static assets from the project assets (we assume the current directory is the directory created in [the first section](#1-download-satportal-artifacts)):

```commandline
cp -r dist/satportal/static/* /var/www/new-portal/public/static
```

There is one caveat.  There is a [js config](https://gitlab.com/SatSenseLtd/satsense-portal/-/blob/master/src/js/config.js) and the value of which almost certainly needs changing (unless releasing to portal-staging).  Because the js code is transpiled and minified, it is not obvious where to make this change.  Open the file (exact name not known before artifact download, hence the wildcard)

```commandline
vi /var/www/new-portal/public/static/js/index-*.js
```

And then search for the string that needs changing (press "/" and type part of the string that needs changing (e.g. "https") and press return).  Edit the string and exit vi.

### 5. Create a virtual environment

Compare the following instructions with the instructions from the [project README](https://gitlab.com/SatSenseLtd/satsense-portal). 

Initialise and activate a virtual environment:

```commandline
cd /var/www/new-portal/virtual_environments
python3.7 -m venv venv
source venv/bin/activate
```

Then change your current directory back to the timestamped directory created in [the first section](#1-download-satportal-artifacts).

Install the requirements:

```commandline
pip install "$(cat dist/satportal/requirements/production.txt | grep numpy)"
pip install GDAL==$(gdal-config --version | awk -F'[.]' '{print $1"."$2}') --global-option=build_ext --global-option="-I/usr/include/gdal"
pip install -r dist/satportal/requirements/production.txt
```

Then install satdom and satportal wheels (the exact file names are indeterminable before downloading the artifacts - hence the wildcards):

```commandline
pip install dist/satdom/satdom-*.whl
pip install dist/satportal/satportal-*.whl
```

Deactivate the virtual environment

```commandline
deactivate
```

## Systemd and gunicorn set up 

Once we've set up the project directories/virtual environment as above, we need something to actually run the application.  This is done via systemd and gunicorn.  We assume the reader has followed the above instructions.

Gunicorn is easy to install via pip. As `flaskuser` then run:
```
source /var/www/new-portal/virtual_environments/venv/bin/activate
pip install gunicorn
```

If you wish, you can check that the site is set up correctly (i.e. that the application will run) by running:

```
gunicorn -b localhost:<port> -w 1 app:app
```
from `/var/www/new-portal/app_home` and replace `<port>` with a port number.

We'd like the above process to be run automatically, which is done by using `systemd`. To set up `systemd`, you'll
need sudo privileges (which `flaskuser` does not have).  First, create a file:
```
sudo touch /etc/systemd/system/newportal.service
sudo chmod 664 /etc/systemd/system/newportal.service
```

Then populate the file, with something similar to following (we've assumed a port of "5000", but be warned this port is already used on the live server):
```
[Unit]
Description = NewPortal
After = network.target

[Service]
User=flaskuser
WorkingDirectory=/var/www/new-portal/app_home
ExecStart=/var/www/new-portal/virtual_environments/venv/bin/gunicorn -b localhost:5000 -w 4 app:app
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
systemctl start newportal
```

We note that the service is named `newportal` due to the name of the systemd file we created. The service can be stopped via:
```
systemctl stop newportal
```

To allow the service to start on server reboot, it needs to be enabled:
```
systemctl enable newportal
```

## Apache set up

Once the portal application is running using systemd.  We need to let the web server (currently apache) know to proxy requests to this service.  To do this we create a virtual host.  For further information about virtual hosts, see the [apache httpd set up](webmisc.md#apache-httpd-set-up).

Here is an example virtual host for the new portal:
```
<VirtualHost *:80>

ServerName newportal.satsense.com
ServerAlias newportal.satsense.com
DocumentRoot /var/www/new-portal/public

# rewrite to https
RewriteEngine on
RewriteCond %{SERVER_NAME} =newportal.satsense.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

</VirtualHost>
<VirtualHost *:443>

ServerName newportal.satsense.com
ServerAlias newportal.satsense.com
DocumentRoot /var/www/new-portal/public

# flask proxy
ProxyPass /static !
ProxyPass /error !
ProxyPass /tiles !
ProxyPass / http://localhost:5000/
ProxyPassReverse / http://localhost:5000/

# logs
ErrorLog /var/www/newportal/log/error.log
CustomLog /var/www/newportal/log/requests.log combined

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
WSGIScriptAlias /tiles /var/www/tileserver-new-ts/Tileserver/tileserver.wsgi
WSGIDaemonProcess tileserver_new_ts python-home=/var/www/venvs/tileserver-new-ts processes=4 threads=1
WSGIProcessGroup tileserver_new_ts

# ssl
SSLCertificateFile /etc/letsencrypt/live/newportal.satsense.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/newportal.satsense.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/newportal.satsense.com/chain.pem

</VirtualHost>
```

In the above:

*   The httpd server listens to `http` requests on port `80`; the configuration inside 
    `<VirtualHost *:80>` is used upon a `http` request to `newportal.satsense.com`.
    The only instruction is to rewrite the request to a `https` request.
*   The httpd server listens to `https` requests on port `443`; the configuration inside 
    `<VirtualHost *:443>` is used upon a `https` request to `newportal.satsense.com`.
*   `DocumentRoot` specifies the directory where the server should look for files.  Without any other 
    configuration, a request to `https://newportal.satsense.com/index.html` would server the file `index.html` 
    inside the `DocumentRoot` directory.
*   The `flask proxy` session allows the httpd server to forward requests to port `5000` (where the flask 
    application is listening).  Values with an exclamation mark (e.g. `ProxyPass /static !`) mean that requests following this pattern should not be proxied (e.g. `https://newportal.satsense.com/static/resource.js`)
*   The `logs` section specifies where the httpd logs should be made.
*   The `headers` section sets headers on each response, these values are recommended to improve the security between the server and the client
*   The `Error Documents` are documents that are served upon a particular error thrown by the server. A `503` corresponds to a `Service Unavailable` and is thrown if the flask server is not running - therefore any website users see a branded error page and not the default httpd error page.
*   The `wsgi` section sets up the tileserver.  Requests to `https://newportal.satsense.com/tiles` are forwarded to the tileserver. See the page about [the tileserver](tileservers.md).
*   The `ssl` section tells the httpd server which ssl certificate to use. **This section is automatically populated by [certbot](webmisc.md#ssl-certificates) and should not be populated manually upon a new release**.

# Release Update

This section deals with releasing an update to a portal that is already live. See [the previous section](#first-release) for a first release of a new portal.  A lot of the steps in this section are similar to first release section - with this in mind, this section lacks the verbosity of the new release section.

## Check and apply database migrations

For more information about alembic (the tool we use for database migrations), see the [satdom README](https://gitlab.com/SatSenseLtd/satsense-domain).

At time of writing the live server has a `satsense_domain` repository at
```
/home/centos/Projects/alembic/satsense-domain
```
which houses a virtual environment called `venv_alembic`.  The `satsense_domain` is installed as editable dependency inside `venv_alembic`.  We use this project to run the alembic migrations. 

Make sure the project is up to date and activate the environment. Change alembic config files (specifically `sqlalchemy.url`).  From directories containing `alembic.ini` config files, run
```
alembic history
alembic current
```
to see current version and alembic history.  Note down what will be updated and check this seems reasonable. If the schema update is just adding new tables/columns (nullable or with default values) then running this migration is unlikely to break anything.  However, if a column name has been changed, or if a column has been deleted then proceed with caution - this needs to be handled carefully and may require more app downtime.

Finally, **if tested and completely happy**, run:
```
alembic upgrade head
```

## Other steps

1.  Log onto api server as flaskuser (can run `sudo su flaskuser` as centos).
2.  Change directory to portal release directory `cd ~/releases/portal`.
3.  Check branch is correct in script `download_portal_artifacts` and run it.
4.  On gitlab, [compare](https://gitlab.com/SatSenseLtd/satsense-portal/-/compare?from=master&to=master) the master branch (if you are releasing the master branch) of `satsense-portal` to the most recent release tag.  Check if the following have been updated:
    * config.ini
    * requirements/production.txt
5. If fields have been added to config.ini, then you will need to add the same fields to `/var/www/new-portal/app_home/config.ini`.  If fields have been deleted, then you can also delete them from the app config file, but it might be safer to do this once the code has been released.
   
The following will actually interact with the current running site.  It is advisable to bring the site down whilst doing these steps - which might be best out of hours.  This can be done via `sudo systemctl stop newportal` which needs to be run as the centos user.  The rest of these instructions assume you are logged on as `flaskuser`.

6. Activate the application virtual environment `source /var/www/new-portal/virtual_environments/venv/bin/activate`.
7. Change directory to the portal artifact directory created in step 3.
9. Uninstall satdom/satportal in the virtual environment `pip uninstall satdom satportal`.
8. If requirements/production.txt had been updated (see step 4), then install the new dependencies `pip install -r dist/satportal/requirements/production.txt`.
9. Reinstall satdom/satsense-portal into the venv:
   ```commandline
   pip install dist/satdom/satdom-*.whl
   pip install dist/satportal/satportal-*.whl
   ```
10. Replace static files in public static
    1. `cd /var/www/new-portal/public/static`
    2. `rm -rv *`
    3. `cp -rv ~/releases/pathtothisrelease/dist/satportal/static/* /var/www/new-portal/public/static`
    4. edit index.js `vi /var/www/new-portal/public/static/js/index.js` and change the tiles url to the correct setting (usually the domain of the portal with the route "tiles", e.g. `https://satshop.satsense.com/tiles`)

Bring the site back up, as centos, run `sudo systemctl start newportal` and check the site works as expected.

# Troubleshooting

| Problem  |  Actions  |
|----------|----------- |
| Web site isn't responding. For example `https://satshop.satsense.com` doesn't respond. |  <ul><li>Check Api server is running</li><li>Check httpd errors</li><li>Check a record</li></ul> |
| Web site reports "Unexpected error occurred" |    <ul><li>Check flask errors</li></ul> |
| Web site reports "Server Unavailable" |    <ul><li>Check gunicorn systemd process is running</li></ul> |
| Web site responds, but logging in does not work | <ul><li>Check flask errors</li><li>Restart flask application</li></ul> |


## Actions

### Check Api server is running

Check you can ssh into the api server.  If not, contact unipart for further help.

### Check a record

Log into cloudflare and check the A record for the domain that you're trying to access.

### Check flask errors

The flask error logs are in the app log directories (locations detailed in the [Live Server Locations](#live-server-locations) section).

### Check httpd errors

The httpd error logs are file called are in the app log directories (locations detailed in the [Live Server Locations](#live-server-locations) section).

### Check gunicorn systemd process is running
```
systemctl status newportal
```
If process is reported as stopped. Then run
```
sudo systemctl start newportal
```

### Restart flask application

Restart the application by stopping and starting:

```
sudo systemctl stop newportal
sudo systemctl start newportal
```