# Tileservers

Instructions for how to install sat-tileserver are kept in the project [README](https://gitlab.com/SatSenseLtd/sat-tileserver/-/blob/master/README.md).  What that README does not make explicit is where everything is installed on the live server.  We detail that in this documentation, along with some caveats when trying to run the tileserver behind apache.

# Contents

* [Live Server Locations](#live-server-locations) - where files related to the tileservers are kept on the live server.
* [First Release](#first-release) - how to release a brand new tileserver.
* [Release Update](#release-update) - how to release an update to tileserver that's already been released.
* [Troubleshooting](#troubleshooting) - something not working? No problem.

# Live Server Locations

On the live server, we run several tileservers, one for each portal project.  This is a list of the tile servers that are running, along with locations of related important files:

| Tileserver name | Domain end point |  Httpd log directory |  Httpd config file |
|---------------|-------------|----------|----------|
| tileserver-satshop | satshop.satsense.com/tiles | /var/www/satshop/log | /etc/httpd/sites-available/satshop.conf |
| tileserver-staging | portal.staging.satsense.com/tiles | /var/www/flask-portal-staging/log | /etc/httpd/sites-available/flask-portal-staging.conf |
| tileserver-demo | demo.satsense.com/tiles | /var/www/demo-portal-live/log | /etc/httpd/sites-available/demo-portal-live.conf |
| tileserver-demo-staging | demo.staging.satsense.com/tiles  | /var/www/demo-portal-staging/log | /etc/httpd/sites-available/demo-portal-staging.conf |
| tileserver-portal2 | portal2.satsense.com/tiles | /var/www/flask-portal2/log | /etc/httpd/sites-available/flask-portal2.conf |
| tileserver-portal3 | portal3.satsense.com/tiles | /var/www/portal3/log | /etc/httpd/sites-available/portal3.conf |

The httpd log directories and httpd config files are shared with the portal httpd set up because they are provided by the same domain. 

Other files related to the above projects are kept in the following locations on the live server:

* Virtual environments are in the directories `/var/www/venvs/<Tileserver name>`.
* Project code is in the directories `/var/www/<Tileserver name>/Tileserver`.
* Tile overview geotiffs are kept in the directory `/data/tile_geotiffs`.  We don't keep an explicit log for where each tileserver's geotiffs are kept. However, one can look in the tileserver config files to see which geotiffs are being used e.g. `/var/www/tileserver-satshop/Tileserver/config.ini` will refer to which geotiffs are being used for `tileserver-satshop`. 


# First Release

These are instructions for if you need to release a new tileserver. If you need to release an update, please see the [Release Update](#release-update) section.

## Set Up Code and Virtual Environment

Create the venvs folder, if it doesn't exist:

```commandline
sudo mkdir /var/www/venvs
sudo chown centos:apache /var/www/venvs
```

Create and activate a new virtual environment for the tileserver:

```commandline
virtualenv /var/www/venvs/tileserver-new-ts
source /var/www/venvs/tileserver-new-ts/bin/activate
```

Make sure mapnik is installed, see the project [README](https://gitlab.com/SatSenseLtd/sat-tileserver/-/blob/master/README.md). Create a new directory for the tileserver project, and checkout the project inside that directory:

```commandline
sudo mkdir /var/www/tileserver-new-ts
sudo chown centos:apache /var/www/tileserver-new-ts
git clone https://gitlab.com/SatSenseLtd/sat-tileserver.git /var/www/tileserver-new-ts/Tileserver
```

Then install the project into the virtual environment as per the project [README](https://gitlab.com/SatSenseLtd/sat-tileserver/-/blob/master/README.md).

Some of the dependencies in the virtual environment will fall fowl of SELinux once the WSGI is set up.  You'll need to update the SELinux context of these dependencies:

```commandline
semanage fcontext -a -t httpd_sys_script_exec_t '/var/www/venvs/tileserver-new-ts/lib/python2.7/site-packages/.*\.so(\..*)?'
sudo restorecon -R -v /var/www/venvs/tileserver-new-ts/lib/python2.7/site-packages/
```

Finally, make sure the project configuration is set up correctly (see the project [README](https://gitlab.com/SatSenseLtd/sat-tileserver/-/blob/master/README.md)).  If you want to test the virtual environment and config is set up correctly, you can run the development server by running

```commandline
python /var/www/tileserver-new-ts/Tileserver/Scripts/run_tilestache_dev_server.py
```

with the `/var/www/venvs/tileserver-new-ts` virtual environment activated. Note you might need to first run

```commandline
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
```

Finally, you will be wanting to serve overview geotiffs from a folder.  With selinux, that folder needs to have the `httpd_sys_content_t` context.  To give an explicit example of how to set this context up, we assume you are storing tile geotiffs within `/data/tile_geotiffs`:

```commandline
sudo semanage fcontext -a -t httpd_sys_content_t "/data/tile_geotiffs(/.*)?"
sudo restorecon -R -v /data/tile_geotiffs
```

## WSGI set up

We currently run the tileserver using the apache module `mod_wsgi`.  Check the folder `/etc/httpd/modules` for a file called `mod_wsgi.so` to see it is installed.  If not, this can be installed via:

```commandline
sudo yum install mod_wsgi
```

By default, mapnik installs in `/usr/local/lib`.  We need to update the library path in apache so that it knows to find it.  This can be set by adding the variable

```
LD_LIBRARY_PATH=/usr/local/lib/
```

to `/etc/sysconfig/httpd`.

Then, a virtual host (e.g. `/etc/httpd/sites-available/new_portal`, full instructions are [here](portal.md)) should be populated with something like:

```commandline
WSGIScriptAlias /tiles /var/www/tileserver-new-ts/Tileserver/tileserver.wsgi
WSGIDaemonProcess tileserver_new_ts python-home=/var/www/venvs/tileserver-new-ts processes=4 threads=1
WSGIProcessGroup tileserver_new_ts
```

Then to load in the new configurations, this requires a `httpd` restart:

```commandline
sudo systemctl restart httpd.service
```

Bear in mind that a httpd restart will mean that the server is unable to respond to requests for a few seconds (therefore best to perform this out of hours).

# Release Update

 These are instructions for if you need to release an update to a tileserver that is already released. 
 
You can update the config as needed, see the project [README](https://gitlab.com/SatSenseLtd/sat-tileserver/-/blob/master/README.md).  To update the code, just use `git` to pull in the latest required changes.  After a change in the project, apache needs to be reloaded:

```commandline
sudo systemctl reload httpd.service
```

This is a soft restart and does not interrupt the current request handling i.e. this can be performed at any point.

# Troubleshooting

If the tileserver is already set up and it suddenly stops working, see the following table:

| Problem  |  Actions  |
|----------|----------- |
| No tiles render |  <ul><li>[Check httpd errors](#check-httpd-errors)</li><li>[Test development server](#test-development-server)</li><li>[Httpd reload or restart](#httpd-reload-or-restart)</li></ul> |
| Overview geotiff tiles render but point tiles do not |  <ul><li>[Check httpd errors](#check-httpd-errors)</li><li>[Test development server](#test-development-server)</li><li>[Httpd reload or restart](#httpd-reload-or-restart)</li></ul> |
| Point tiles render, but overview geotiff tiles do not |   <ul><li>[Check tileserver config vs overview geotiffs](#check-tileserver-config-vs-overview-geotiffs)</li><li>[Check httpd errors](#check-httpd-errors)</li><li>[Test development server](#test-development-server)</li><li>[Httpd reload or restart](#httpd-reload-or-restart)</li></ul> |

Some top tips for when setting up the tileserver and you run into problems.  Check the logs and see if the error matches with the following:

| Problem  |  Actions  |
|----------|----------- |
|  `ImportError: No module named Image`  | Check the virtual environment `.so` files have the correct selinux context, see above about adding `httpd_sys_script_exec_t` context |
|  `NameError: global name 'mapnik' is not defined`  | Check you've added `LD_LIBRARY_PATH=/usr/local/lib/` to `/etc/sysconfig/httpd`. |
|  `/data/tile_geotiffs/.../path/to/geotiff/tiles_4326_10.tif does not exist in file system`  | Check the geotiff files have the correct selinux context, see above about adding `httpd_sys_content_t` context |


## Actions

These are the instructions for the actions list above, we try to provide generic instructions but for illustrate them with examples of commands specific to the satshop tileserver (i.e. the tileserver named "tileserver-satshop").

### Check httpd errors

The httpd error logs are in the httpd log directories (locations detailed in the [Live Server Locations](#live-server-locations) section).


### Test development server

This might give you more insights into the errors that are being thrown.

Activate the tileserver virtual environment
```commandline
source /var/www/venvs/tileserver-satshop/bin/activate
```

Change directory to the tileserver root
```commandline
cd /var/www/tileserver-satshop/Tileserver
```

And run the development server:

```commandline
python Scripts/run_tilestache_dev_server.py -p <port of your choosing>
```

If the server runs without errors, you may want to check it can receive requests successfully. You can set this up using a proxy in apache, which will forward the requests to this port (similar to what we do with the flask applications).  **Do not** leave the application running in this state - the above command just runs a development server and it should not be used in production.

### Check tileserver config vs overview geotiffs

Check where the config (locations are in the root of the Project code, see [Live Server Locations](#live-server-locations) section) is expecting to find the geotiff overview tiles, make sure they exist/replace with back ups.

### Httpd reload or restart

Reloading is a softer version of the "restarting" the httpd process. Reloading should not impact the server dealing with requests, but restarting may. The commands are:

```
sudo systemctl reload httpd.service
sudo systemctl restart httpd.service
```
