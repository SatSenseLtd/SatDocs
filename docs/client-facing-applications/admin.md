# Admin Suite (satadmin)

This is a web application with two main responsibilies:

* allow the management of users on the portal/satshop and api.
* monitor/control the processing of SatSense data.

Instructions for how to install an instance of the admin suite into a virtual environment are kept in the project [README](https://gitlab.com/SatSenseLtd/satsense-admin/-/blob/master/README.md).  What that README does not make explicit is where everything is installed on the live server.  We detail that in this documentation, along with how to set up apache to forward on requests.

# Contents

* [Live Server Locations](#live-server-locations) - where files related to the admin suite are kept on the live server.
* [First Release](#first-release) - how to release a brand new admin suite.
* [Release Update](#release-update) - how to release an update to an admin suite that's already been released.
* [Troubleshooting](#troubleshooting) - something not working? No problem.

# Live Server Locations

On the live server, we run several portals and in turn we therefore run several admin suites (which deal with the separate user databases). This is a list of the admin suites that are running, along with locations of related important files:

| Admin instance name |  Domain end point | Associated portal name | Project directory |  Httpd config file | Systemd file |
|---------------|-------------|----------|----------| -------- | -------- |
| satshop admin | admin.satsense.com | satshop | /var/www/admin-live | /etc/httpd/sites-available/admin-live.conf | /etc/systemd/system/adminlive.service |
| staging admin | admin.staging.satsense.com | staging | /var/www/admin-staging |  /etc/httpd/sites-available/admin-staging.conf | /etc/systemd/system/adminstaging.service |
| portal2 admin | portal2.satsense.com/admin | portal2 |  /var/www/admin2 | shared with portal2 i.e. /etc/httpd/sites-available/flask-portal2.conf | /etc/systemd/system/admin2.service |
| portal3 admin| portal3.satsense.com/admin | portal3 |  /var/www/admin3 | shared with portal3 i.e. /etc/httpd/sites-available/portal3.conf | /etc/systemd/system/admin3.service |

Notice that "portal2 admin" and "portal3 admin" share a domain with portal2 and portal3 respectively - we realised that it was easier to share this domain rather than set up two domains every single time we want a new portal.  "satshop admin" and "staging admin" only use their dedicated domains for historical purposes. 

## Project directory structure 

The structure of each "Project directory" is as follows:

```
- app_home
- log
    - flask   
- public 
    - error
- virtual_environments
```

Compare this with the [directory structure of the satshop/portal](portal.md#project-directory-structure). Notice that the only difference is the lack of `public/static` directory - we allow the flask process to serve static assets instead of the apache server.  There is no reason for this other than the time taken to set this up, this can easily be set up if serving assets via flask becomes an issue. 

Although we describe the purpose of each directory in the portal docs, we do the same again here for completeness.  The app is installed into a virtual environment inside `virtual_environments`. The directory `app_home` contains bootstrap code needed for the application, namely `app.py` and `config.ini` (see [project root](https://gitlab.com/SatSenseLtd/satsense-admin)). 

The directory `public/error` houses static error files; when releasing, it is common to bring the flask server down, at which point apache will return the `503` error document in this directory.

The `log` directory contains logs for the application.  Any errors/requests that the apache server runs into can be found in this directory.  The flask application itself is also configured to report any errors, these can be found in the subdirectory `log/flask`. 

# First Release

These are instructions if you need to release a new admin suite.  The reader will also need to set up any ssl certificates and `A` records. See the [web miscellaneous docs](webmisc.md) for more detail.   If you need to release an update, please see the [Release Update](#release-update) section.  We assume user/processing databases are set up already.

## Set up project directory

In the following, we use a location of a "new-admin".  The reader should change the folder paths for the application they are setting up.

If this is the first flask application on the server, it is best to add a user to handle the flask applications:
```
useradd flaskuser
```

Now we make the directories as per the [Project directory structure](#project-directory-structure) section:

```
sudo mkdir /var/www/new-admin/
cd /var/www/new-admin/

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

Notice that `flaskuser` is the owner of all the directories, but that apache needs access to the `public` and `logs` directories.  The former is so that the apache server can serve files from this directory without having to give send a proxy request to the flask application, if for example, serving the `503` page. The latter is so the apache server can write logs to this directory. To write logs, we have to also give that folder SELinux permissions to do that: 

```
sudo semanage fcontext -a -t httpd_log_t "/var/www/new-admin/log(/.*)?"
sudo restorecon -R -v log/
```

The directories are now set up ready to be populated.

## Populate project directories

We will need to:

1. Download artifacts from the build stage of a [satsense admin gitlab pipeline](https://gitlab.com/SatSenseLtd/satsense-admin/-/pipelines).
2. Populate the `app_home` directory.
3. Create a 503 document in `public/error`.
4. Create a virtual environment for the application and install the project.

We run through each of these steps in succession.  Unless otherwise stated, we will assume that the following statements will be run as `flaskuser`.

### 1. Download satadmin artifacts

We keep a shell script on the live server that downloads the artifacts automatically for us.  This is kept in the file:

```
/home/flaskuser/releases/admin/download_admin_artifacts
```

We keep a copy here as well:

```shell
#!/bin/bash

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
CURL_OUTFILE="satadmin_artifacts_$TIMESTAMP.zip"
SATADMIN_BRANCH="master"
HTTP_CODE=$(curl --write-out "%{http_code}" --location --output $CURL_OUTFILE --header "PRIVATE-TOKEN: <gitlab ro access token>" "https://gitlab.com/api/v4/projects/16546505/jobs/artifacts/$SATADMIN_BRANCH/download?job=create_artifacts")
NEW_DIR="satadmin_$TIMESTAMP"
mkdir $NEW_DIR
unzip $CURL_OUTFILE -d $NEW_DIR
rm $CURL_OUTFILE
printf "Response code: $HTTP_CODE\nOut Dir: $NEW_DIR\n"
```

Note that if you use the code above, you should create a [gitlab access token](https://gitlab.com/-/profile/personal_access_tokens) (preferably read only) and replace `<gitlab ro access token>` with it. However, the file on the server has an access token pre-filled.

Further, if you need to release a branch other than `master`, then you can change the `SATADMIN_BRANCH` variable.

By convention, we run the `download_admin_artifacts` file from inside its directory, i.e. 

```commandline
cd /home/flaskuser/releases/admin
./download_admin_artifacts
```

and this will create a timestamped directory of the needed artifacts for a satadmin release.

### 2. Populate app_home

We assume that the current directory is the timestamped directory created in [the first section](#1-download-satadmin-artifacts)

```commandline
cp dist/satadmin/app.py /var/www/new-admin/app_home
cp dist/satadmin/config.ini /var/www/new-admin/app_home
```

Here, `app.py` is the entry point for the application and `config.ini` is a config file. The latter needs to be edited with config settings for the application (e.g. database strings, live api keys etc).


### 3. Create a 503 document

Create the required error folder:

```commandline
mkdir /var/www/new-admin/public/error
```

And copy the 503 document from the admin artifacts (we assume the current directory is the directory created in [the first section](#1-download-satadmin-artifacts)):

```commandline
cp dist/satadmin/error/* /var/www/new-admin/public/error
```

### 4. Create a virtual environment

Compare the following instructions with the instructions from the [project README](https://gitlab.com/SatSenseLtd/satsense-admin). 

Initialise and activate a virtual environment:

```commandline
cd /var/www/new-admin/virtual_environments
python3.7 -m venv venv
source venv/bin/activate
```

Then change your current directory back to the timestamped directory created in [the first section](#1-download-satadmin-artifacts).

Install the requirements:

```commandline
pip install "$(cat dist/satadmin/requirements/production.txt | grep numpy)"
pip install GDAL==$(gdal-config --version | awk -F'[.]' '{print $1"."$2}') --global-option=build_ext --global-option="-I/usr/include/gdal"
pip install -r dist/satadmin/requirements/production.txt
```

Then install satdom and satadmin wheels (the exact file names are indeterminable before downloading the artifacts - hence the wildcards):

```commandline
pip install dist/satdom/satdom-*.whl
pip install dist/satadmin/satadmin-*.whl
```

Deactivate the virtual environment

```commandline
deactivate
```

## Systemd and gunicorn set up 

Once we've set up the project directories/virtual environment as above, we need something to actually run the application.  This is done via systemd and gunicorn.  We assume the reader has followed the above instructions.

Gunicorn is easy to install via pip. As `flaskuser` then run:
```
source /var/www/new-admin/virtual_environments/venv/bin/activate
pip install gunicorn
```

If you wish, you can check that the site is set up correctly (i.e. that the application will run) by running:

```
gunicorn -b localhost:<port> -w 1 app:app
```
from `/var/www/new-admin/app_home` and replace `<port>` with a port number.

We'd like the above process to be run automatically, which is done by using `systemd`. To set up `systemd`, you'll
need sudo privileges (which `flaskuser` does not have).  First, create a file:
```
sudo touch /etc/systemd/system/newadmin.service
sudo chmod 664 /etc/systemd/system/newadmin.service
```

Then populate the file, with something similar to following (we've assumed a port of "5000", but be warned this port is already used on the live server):
```
[Unit]
Description = NewAdmin
After = network.target

[Service]
User=flaskuser
WorkingDirectory=/var/www/new-admin/app_home
ExecStart=/var/www/new-admin/virtual_environments/venv/bin/gunicorn -b localhost:5000 -w 4 app:app
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
systemctl start newadmin
```

We note that the service is named `newadmin` due to the name of the systemd file we created. The service can be stopped via:
```
systemctl stop newadmin
```

To allow the service to start on server reboot, it needs to be enabled:
```
systemctl enable newadmin
```

## Apache set up

Once the admin application is running using systemd.  We need to let the web server (currently apache) know to proxy requests to this service.  To do this we create a virtual host.  For further information about virtual hosts, see the [apache httpd set up](webmisc.md#apache-httpd-set-up).

Here is an example virtual host for the new admin suite:
```
<VirtualHost *:80>

ServerName newadmin.satsense.com
ServerAlias newadmin.satsense.com
DocumentRoot /var/www/new-admin/public

# rewrite to https
RewriteEngine on
RewriteCond %{SERVER_NAME} =newadmin.satsense.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

</VirtualHost>
<VirtualHost *:443>

ServerName newadmin.satsense.com
ServerAlias newadmin.satsense.com
DocumentRoot /var/www/new-admin/public

# flask proxy
ProxyPass /error !
ProxyPass / http://localhost:5000/
ProxyPassReverse / http://localhost:5000/

# logs
ErrorLog /var/www/newadmin/log/error.log
CustomLog /var/www/newadmin/log/requests.log combined

# headers:
Header always set Strict-Transport-Security "max-age=31536000; includeSubdomains"
Header always set X-Frame-Options "deny"
Header always set X-Xss-Protection "1; mode=block"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "no-referrer"
Header always edit Set-Cookie ^(.*)$ $1;HttpOnly;Secure;SameSite=Strict

# Error Documents
ErrorDocument 503 /error/503.html

# ssl
SSLCertificateFile /etc/letsencrypt/live/newadmin.satsense.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/newadmin.satsense.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/newadmin.satsense.com/chain.pem

</VirtualHost>
```

In the above:

*   The httpd server listens to `http` requests on port `80`; the configuration inside 
    `<VirtualHost *:80>` is used upon a `http` request to `newadmin.satsense.com`.
    The only instruction is to rewrite the request to a `https` request.
*   The httpd server listens to `https` requests on port `443`; the configuration inside 
    `<VirtualHost *:443>` is used upon a `https` request to `newadmin.satsense.com`.
*   `DocumentRoot` specifies the directory where the server should look for files.  Without any other 
    configuration, a request to `https://newadmin.satsense.com/index.html` would server the file `index.html` 
    inside the `DocumentRoot` directory.
*   The `flask proxy` session allows the httpd server to forward requests to port `5000` (where the flask application is listening).  Values with an exclamation mark (e.g. `ProxyPass /error !`) mean that requests following this pattern should not be proxied (e.g. `https://newadmin.satsense.com/error/503.html`)
*   The `logs` section specifies where the httpd logs should be made.
*   The `headers` section sets headers on each response, these values are recommended to improve the security between the server and the client
*   The `Error Documents` are documents that are served upon a particular error thrown by the server. A `503` corresponds to a `Service Unavailable` and is thrown if the flask server is not running - therefore any website users see a branded error page and not the default httpd error page.
*   The `ssl` section tells the httpd server which ssl certificate to use. **This section is automatically populated by [certbot](webmisc.md#ssl-certificates) and should not be populated manually upon a new release**.

# Release Update

This section deals with releasing an update to an admin suite that is already live. See [the previous section](#first-release) for a first release of a new admin suite.  A lot of the steps in this section are similar to first release section - with this in mind, this section lacks the verbosity of the new release section.

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
2.  Change directory to admin release directory `cd ~/releases/admin`.
3.  Check branch is correct in script `download_admin_artifacts` and run it.
4.  On gitlab, [compare](https://gitlab.com/SatSenseLtd/satsense-admin/-/compare?from=master&to=master) the master branch (if you are releasing the master branch) of `satsense-admin` to the most recent release tag.  Check if the following have been updated:
    * config.ini
    * requirements/production.txt
5. If fields have been added to config.ini, then you will need to add the same fields to `/var/www/new-admin/app_home/config.ini`.  If fields have been deleted, then you can also delete them from the app config file, but it might be safer to do this once the code has been released.
   
The following will actually interact with the current running site.  It is advisable to bring the site down whilst doing these steps - which might be best out of hours.  This can be done via `sudo systemctl stop newadmin` which needs to be run as the centos user.  The rest of these instructions assume you are logged on as `flaskuser`.

6. Activate the application virtual environment `source /var/www/new-admin/virtual_environments/venv/bin/activate`.
7. Change directory to the admin artifact directory created in step 3.
9. Uninstall satdom/satadmin in the virtual environment `pip uninstall satdom satadmin`.
8. If requirements/production.txt had been updated (see step 4), then install the new dependencies `pip install -r dist/satadmin/requirements/production.txt`.
9. Reinstall satdom/satsense-admin into the venv:
   ```commandline
   pip install dist/satdom/satdom-*.whl
   pip install dist/satadmin/satadmin-*.whl
   ```
   
Bring the site back up, as centos, run `sudo systemctl start newadmin` and check the site works as expected.

# Troubleshooting

| Problem  |  Actions  |
|----------|----------- |
| Web site isn't responding. For example `https://admin.satsense.com` doesn't respond. |  <ul><li>Check Api server is running</li><li>Check httpd errors</li><li>Check a record</li></ul> |
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
systemctl status newadmin
```
If process is reported as stopped. Then run
```
sudo systemctl start newadmin
```

### Restart flask application

Restart the application by stopping and starting:

```
sudo systemctl stop newadmin
sudo systemctl start newadmin
```