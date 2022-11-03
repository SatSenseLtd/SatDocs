# Overview

This page includes documentation for the satsense API.  Similarly to the [portal](portal.md), the API
is written using [flask](https://flask.palletsprojects.com/en/1.1.x/).  Whilst some details in the portal documentation
are similar, the projects are quite different in design style for the following reasons:

*  the API response contents are all json; there is no need for html, css or js (i.e. the templates/static/src
   directories);
*  the API uses the [flask-restful](https://flask-restful.readthedocs.io/en/latest/) extension, which is quite different
   syntactically to vanilla flask.

Instructions for how to install an instance of the portal/satshop into a virtual environment are kept in the project [README](https://gitlab.com/SatSenseLtd/satsense-api/-/blob/master/README.md).  What that README does not make explicit is where everything is installed on the live server.  We detail that in this documentation, along with how to set up apache to forward on requests.

# Contents

* [Live Server Locations](#live-server-locations) - where files related to the api are kept on the live server.
* [First Release](#first-release) - how to release a brand new api.
* [Release Update](#release-update) - how to release an update to an api that's already been released.
* [Troubleshooting](#troubleshooting) - something not working? No problem.

# Live Server Locations

On the live server, we run a few apis; one live api and a couple of test apis. This is a list of the apis that are running, along with locations of related important files:

| Api instance name | Domain end point |  Project directory |  Httpd config file | Systemd file | Celery systemd file |
|---------------|-------------|----------|----------| -------- | -------- |
| live | api.satsense.com | /var/www/api-live | /etc/httpd/sites-available/api-live.conf | /etc/systemd/system/apilive.service | /etc/systemd/system/apilivecelery.service |
| test | api.test.satsense.com | /var/www/api-test |  /etc/httpd/sites-available/api-test.conf | /etc/systemd/system/apitest.service | /etc/systemd/system/apitestcelery.service |
| staging | api.staging.satsense.com | /var/www/api-staging | /etc/httpd/sites-available/api-staging.conf | /etc/systemd/system/apistaging.service  | /etc/systemd/system/apistagingcelery.service |

## Project directory structure

Note that the directory structure is similar to that of the [portal site](portal.md#project-directory-structure), and although it may seem like this section repeats some of that documentation, we choose to be explicit here.

Each "Project directory" (see table above) has the following structure:

```
- app_home
- celery
- log   
- virtual_environments
```

The app is installed into a virtual environment inside `virtual_environments`. The directory `app_home` contains

* bootstrap code needed for the applications, namely `app.py`, `celery_app.py` and `config.ini`.
* a config file for gunicorn; `gunicorn_config.py`.
* a config file for new relic (a web based monitoring/metric tool) - `newrelic.ini`.

The `log` directory contains logs for application.  Any errors/requests that the apache server runs into can be found in this directory.  The flask application itself is also configured to report any errors, these can be found in the subdirectory `log/flask`.

The `celery` directory contains:

* a config for celery; `celery_env`.
* `pid` files to help monitor when the app is running.
* logs for the celery application.

# Access To Apis

Access to the apis is controlled by two elements, these are:

* the user having an api key
* the user having access to "api resources"

The idea is that a user is added to the resources they need access to - the current resources are:

* report - referring to groundsure end points
* vels_ts - referring to obtaining velocities and time series data via an api

A user with access to either of these resources can use an api key that's attributed to them.  Or if they have access to both, the same api key will give them access to both resources.

To give a user access to the live api, this can be done from the "view user" page on https://admin.satsense.com.  Click on "View User Api Details" and add the user to resources and create them an api key.

For the other apis (namely the "test" and "staging" apis), there may not be an admin suite associated with the user database for these apis.  You can give access to these apis by running the following sql commands against a user database:

```
INSERT INTO insar_api_keys(user_id, api_key) VALUES (<user id>, '<new api key>');
UPDATE portal_users SET api_resources = ARRAY['reports'] WHERE id = <user id>;
```

Replace `<user id>` with the id of the user you'd like to give permissions to and replace `<new api key>` with a suitable api key.  Obviously, you can also give the user access to the `vel_ts` resource by changing the update statement accordingly.  Note in the code, we just use a uuid4 to generate an api key so you could just run `python -c "from uuid import uuid4; print(uuid4())"` to generate an ad hoc api key.

A final note that at the time of writing, the staging api uses the user database `staging users` and the test api users the user database `api_test_users`.

# First Release

In the following, we detail how to set up the initial release for the api. The reader will also need to set up any ssl certificates and `A` records. See the [web miscellaneous docs](webmisc.md) for more detail.  If you need to release an update to an api that is already live, then please see the [Release Update](#release-update) section. In the following, we assume that user/InSAR databases are set up already.

## Set up project directory

In the following, we use a location of a "new-api".  The reader should change the folder paths for the application they are setting up.

If this is the first application on the server, it is best to add a user to handle the flask applications:
```
useradd flaskuser
```

Now we make the directories as per the section:

```
sudo mkdir /var/www/new-api/
cd /var/www/new-api/

sudo mkdir log/
sudo chown flaskuser:apache log/

sudo mkdir log/flask
sudo chown flaskuser:flaskuser log/flask/

sudo mkdir log/gunicorn
sudo chown flaskuser:flaskuser log/gunicorn/

sudo mkdir app_home/
sudo chown flaskuser:flaskuser app_home/

sudo mkdir virtual_environments/
sudo chown flaskuser:flaskuser virtual_environments/

sudo mkdir celery/
sudo chown flaskuser:flaskuser celery/
```

Notice that `flaskuser` is the owner of all the directories, but that apache needs access to the `log` directories.  To write logs, we have to also give that folder SELinux permissions to do that:

```
sudo semanage fcontext -a -t httpd_log_t "/var/www/new-api/log(/.*)?"
sudo restorecon -R -v log/
```

The directories are now set up ready to be populated.

## Populate project directories

We will need to:

1. Download artifacts from the build stage of a [satapi gitlab pipeline](https://gitlab.com/SatSenseLtd/satsense-portal/-/pipelines).
2. Populate the `app_home` directory.
3. Create a virtual environment for the application and install the project.

We run through each of these steps in succession.  Unless otherwise stated, we will assume that the following statements will be run as `flaskuser`.

### 1. Download artifacts

We keep a shell script on the live server that downloads the artifacts automatically for us.  This is kept in the file:

```
/home/flaskuser/releases/api/download_api_artifacts
```

We keep a copy here as well:

```shell
#!/bin/bash

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
CURL_OUTFILE="satapi_artifacts_$TIMESTAMP.zip"
SATAPI_BRANCH="master"
HTTP_CODE=$(curl --write-out "%{http_code}" --location --output $CURL_OUTFILE --header "PRIVATE-TOKEN: <gitlab ro access token>" "https://gitlab.com/api/v4/projects/11778627/jobs/artifacts/$SATAPI_BRANCH/download?job=create_artifacts")
NEW_DIR="satapi_$TIMESTAMP"
mkdir $NEW_DIR
unzip $CURL_OUTFILE -d $NEW_DIR
rm $CURL_OUTFILE
printf "Response code: $HTTP_CODE\nOut Dir: $NEW_DIR\n"
```

Note that if you use the code above, you should create a [gitlab access token](https://gitlab.com/-/profile/personal_access_tokens) (preferably read only) and replace `<gitlab ro access token>` with it. However, the file on the server has an access token pre-filled.

Further, if you need to release a branch other than `master`, then you can change the `SATAPI_BRANCH` variable.

By convention, we run the `download_api_artifacts` file from inside its directory, i.e.

```commandline
cd /home/flaskuser/releases/api
./download_api_artifacts
```

and this will create a timestamped directory of the needed artifacts for a satapi release.

### 2. Populate app_home

We assume that the current directory is the timestamped directory created in [the first section](#1-download-artifacts)

```commandline
cp dist/satapi/app.py /var/www/new-api/app_home
cp dist/satapi/celery_app.py /var/www/new-api/app_home
cp dist/satapi/config.ini /var/www/new-api/app_home
```

Here, `app.py` and `celery_app.py` are the entry points for the applications. The file `config.ini` is a config file for the applications. This config file needs to be edited with config settings for the application (e.g. database strings, etc).  See more information about the config file in the [project README](https://gitlab.com/SatSenseLtd/satsense-api).

### 3. Create virtual environment

Compare the following instructions with the instructions from the [project README](https://gitlab.com/SatSenseLtd/satsense-api).

Initialise and activate a virtual environment:

```
cd /var/www/new-api/virtual_environments
/usr/local/bin/python3.7 -m venv venv
source venv/bin/activate
```

Then change your current directory back to the timestamped directory created in [the first section](#1-download-artifacts).

Install the requirements:

```
pip install "$(cat dist/satapi/requirements/production.txt | grep numpy)"
pip install GDAL==$(gdal-config --version | awk -F'[.]' '{print $1"."$2}') --global-option=build_ext --global-option="-I/usr/include/gdal"
pip install -r dist/satapi/requirements/production.txt
```

If you run into trouble installing psycopg2 with an error that pg_config could not be found, specify the postgres installation path.  On staging this would be:
```
export PATH=/usr/pgsql-13/bin/:$PATH

```
Make sure all requirements install without error before continuing.

Then install satdom and satapi wheels (the exact file names are indeterminable before downloading the artifacts - hence the wildcards):

```commandline
pip install dist/satdom/satdom-*.whl
pip install dist/satapi/satapi-*.whl
```

Deactivate the virtual environment

```
deactivate
```

## systemd and gunicorn set up

Install gunicorn inside the virtual environment
```
source /var/www/new-api/virtual_environments/venv/bin/activate
pip install gunicorn
```

If you wish, you can check that the api is set up correctly (i.e. that the application will run) by running:

```
gunicorn -b localhost:<port> -w 1 app:app
```
from `/var/www/new-api/app_home` and replace `<port>` with a port number.

As discussed, the api actually requires two processes to be run; the `app` which handles the requests, and the `celery_app` process that handles asynchronous processes.  We would like to set both of these processes to be run automatically, which is done by using `systemd`.

#### set up for `app`

To set up `systemd`, you'll need sudo privileges (which `flaskuser` does not have).  First, create a file:
```
sudo touch /etc/systemd/system/newapi.service
sudo chmod 664 /etc/systemd/system/newapi.service
```

Then populate this file, here is an example config

```
[Unit]
Description = NewApi
After = network.target

[Service]
User=flaskuser
Environment="APP_HOME_DIR=/var/www/new-api/app_home"
Environment="APP_VENV_DIR=/var/www/new-api/virtual_environments/venv"
WorkingDirectory=/var/www/new-api/app_home
ExecStart=/bin/sh -c '${APP_VENV_DIR}/bin/gunicorn -c ${APP_HOME_DIR}/gunicorn_config.py app:app'
Restart=always

[Install]
WantedBy = multi-user.target
```

Notice that this refers to a config file `gunicorn_config.py` in `/var/www/new-api/app_home`, we also need to create that file. Here is an example of the expected contents of that file:
```
bind = "localhost:5000"

workers = 4

accesslog = "/var/www/new-api/log/gunicorn/accesslog"
errorlog = "/var/www/new-api/log/gunicorn/errorlog"
loglevel = "info"
```

The service can be started via:
```
systemctl start newapi
```

or stopped via:
```
systemctl stop newapi
```

To allow the service to start on server reboot, it needs to be enabled:
```
systemctl enable newapi
```

#### set up for `celery_app`

First, create a file:
```
sudo touch /etc/systemd/system/newapicelery.service
sudo chmod 664 /etc/systemd/system/newapicelery.service
```

Then populate this file, here is an example config

```
[Unit]
Description=NewApiCelery
After=network.target

[Service]
Type=forking
User=flaskuser
Group=flaskuser
EnvironmentFile=/var/www/new-api/celery/celery_env
WorkingDirectory=/var/www/new-api/app_home
ExecStart=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi start $CELERYD_NODES  --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
ExecStop=/bin/sh -c '${CELERY_BIN} multi stopwait $CELERYD_NODES --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} --loglevel="${CELERYD_LOG_LEVEL}"'
ExecReload=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi restart $CELERYD_NODES --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
Restart=always

[Install]
WantedBy=multi-user.target
```

Notice that this config refers to a file containing environment varibles in the location `/var/www/new-api/celery/celery_env`.  An example of the contents of this file is:

```
CELERYD_NODES="newapiw1"
CELERY_BIN="/var/www/new-api/virtual_environments/venv/bin/celery"
CELERY_APP="celery_app"
CELERYD_OPTS="--time-limit=300 --concurrency=1"
CELERYD_PID_FILE="/var/www/new-api/celery/%n.pid"
CELERYD_LOG_FILE="/var/www/new-api/celery/%n%I.log"
CELERYD_LOG_LEVEL="INFO"
```

See the [celery documentation](https://docs.celeryproject.org/en/stable/userguide/daemonizing.html#usage-systemd) for more information.

The service can be altered similarly to the above, e.g.
```
systemctl start newapicelery
systemctl stop newapicelery
systemctl enable newapicelery
```

## Apache set up

Once the api application is running using systemd, we need to let the web server (currently apache) know to proxy requests to this service.  To do this we create a virtual host.  For further information about virtual hosts, see the [apache httpd set up](webmisc.md#apache-httpd-set-up).

Here is an example virtual host for the new api:
```
<VirtualHost *:80>

ServerName newapi.satsense.com
ServerAlias newapi.satsense.com

RewriteEngine on
RewriteCond %{SERVER_NAME} =newapi.satsense.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]

</VirtualHost>
<VirtualHost *:443>

ServerName newapi.satsense.com
ServerAlias newapi.satsense.com

# flask proxy
ProxyPass / http://localhost:5000/
ProxyPassReverse / http://localhost:5000/

# logs
ErrorLog /var/www/new-api/log/error.log
CustomLog /var/www/new-api/log/requests.log combined

# headers:
Header always set Strict-Transport-Security "max-age=300; includeSubdomains"
Header always set X-Frame-Options "deny"

#s sl
SSLCertificateFile /etc/letsencrypt/live/newapi.satsense.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/newapi.satsense.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/newapi.satsense.com/chain.pem
</VirtualHost>
```

In the above:

*   The httpd server listens to `http` requests on port `80`; the configuration inside `<VirtualHost *:80>` is used upon a `http` request to `newapi.satsense.com`. The only instruction is to rewrite the request to a `https` request.
*   The httpd server listens to `https` requests on port `443`; the configuration inside `<VirtualHost *:443>` is used upon a `https` request to `newapi.satsense.com`.
*   `DocumentRoot` specifies the directory where the server should look for files.  Without any other configuration, a request to `https://newapi.satsense.com/index.html` would server the file `index.html` inside the `DocumentRoot` directory.
*   The `flask proxy` session allows the httpd server to forward requests to port `5000` (where the flask application is listening).
*   The `logs` section specifies where the httpd logs should be made.
*   The `headers` section sets headers on each response, these values are recommended to improve the security between the server and the client
*   The `ssl` section tells the httpd server which ssl certificate to use. **This section is automatically populated by [certbot](webmisc.md#ssl-certificates) and should not be populated manually upon a new release**.

# Release Update

This section deals with releasing an update to an api that is already live. See [the previous section](#first-release) for a first release of a new api.  A lot of the steps in this section are similar to first release section - with this in mind, this section lacks the verbosity of the new release section.

First you might want to make back ups of the current project files that will change.  For the staging api, as `flaskuser` you can run

```
cp -r /var/www/api-staging/virtual_environments/venv ~/path/to/backup
```

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
2.  Change directory to API release directory `cd ~/releases/api`.
3.  Check branch is correct in script `download_api_artifacts` and run it.
4.  On gitlab, [compare](https://gitlab.com/SatSenseLtd/satsense-api/-/compare?from=master&to=master) the master branch (if you are releasing the master branch) of `satsense-api` to the most recent release tag.  Check if the following have been updated:
    * config.ini
    * requirements/production.txt
5. If fields have been added to config.ini, then you will need to add the same fields to `/var/www/new-api/app_home/config.ini`.  If fields have been deleted, then you can also delete them from the app config file, but it might be safer to do this once the code has been released.

The following will actually interact with the current running site.  It is advisable to bring the api and celery process down whilst doing these steps - which might be best out of hours.  This can be done via
```
sudo systemctl stop newapi
sudo systemctl stop newapicelery
```
which needs to be run as the centos user.  The rest of these instructions assume you are logged on as `flaskuser`.

6. Activate the application virtual environment `source /var/www/new-api/virtual_environments/venv/bin/activate`.
7. Change directory to the API artifact directory created in step 3.
9. Uninstall satdom/satapi in the virtual environment `pip uninstall satdom satapi`.
8. If requirements/production.txt had been updated (see step 4), then install the new dependencies `pip install -r dist/satapis/requirements/production.txt`.
9. Reinstall satdom/satsense-api into the venv:
   ```commandline
   pip install dist/satdom/satdom-*.whl
   pip install dist/satapi/satapi-*.whl
   ```

Bring the apps back up, as centos, run
```
sudo systemctl start newapi
sudo systemctl start newapicelery
```
and check the api/celery process works as expected.

# Troubleshooting

| Problem  |  Actions  |
|----------|----------- |
| `https://api.satsense.com` doesn't respond. |  <ul><li>Check Api server is running</li><li>Check httpd errors</li><li>Check a record</li></ul> |
| Api reports "Unexpected error occurred" |    <ul><li>Check flask errors</li></ul> |
| Api reports "Server Unavailable" |    <ul><li>Check gunicorn systemd process is running</li></ul> |

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
systemctl status newapi
```
If process is reported as stopped. Then run
```
sudo systemctl start newapi
```
