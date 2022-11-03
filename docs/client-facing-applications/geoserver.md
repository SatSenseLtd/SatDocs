# Overview

### Contents:
* [Installation and Configuration](#installation-and-configuration)
    * [Install Tomcat](#install-tomcat)
    * [Install GeoServer](#install-geoserver)
    * [Enable Reverse Proxy](#enable-proxy-from-apache-httpd)
    * [GeoServer Configuration](#geoserver-configuration)
* [Using Geoserver](#using-geoserver)

### Useful Resources

* [Official User Manual](https://docs.geoserver.org/latest/en/user/)
* [Geo-Solutions Training Modules](https://geoserver.geo-solutions.it/edu/en/)

# Installation and Configuration

We choose to install GeoServer using the [web archive](https://docs.geoserver.org/latest/en/user/installation/war.html) 
via a [tomcat container](https://tomcat.apache.org/).  This seems to be recommended by some users in the community above 
using the [command line installation](https://docs.geoserver.org/latest/en/user/installation/linux.html) which utilises 
an in built [jetty container](http://www.eclipse.org/jetty/).

We first provide instructions for installing tomcat, and then for installing GeoServer itself.  Finally, we add some
notes about configuring GeoServer before it should be used.

## Install Tomcat

We use [this linuxize post](https://linuxize.com/post/how-to-install-tomcat-9-on-centos-7/) for reference, but make some
changes - for instance, we use java 11 and not 8.

First make sure the yum packages are up to date:

```shell script
sudo yum update
```

First we make sure that we've installed a Java Development Kit.  One can us the openjdk version:

```shell script
sudo yum install java-11-openjdk-devel
```

We add a tomcat user and log in:

```shell script
sudo useradd tomcat
sudo su tomcat
```

Get the latest download url from the tomcat [download page](https://tomcat.apache.org/download-90.cgi), by right 
clicking on the core `tar.gz` link and copying it.  Note the linked page is for tomcat 9, there may be later version 
available.  Once you have the url, `wget` the `tar.gz` file and unpack it: 

```shell script
# change the following with your url!
wget https://mirrors.ukfast.co.uk/sites/ftp.apache.org/tomcat/tomcat-9/v9.0.36/bin/apache-tomcat-9.0.36.tar.gz
tar -xf apache-tomcat-9.0.36.tar.gz
```

Now create a symbolic link so that it is easy to upgrade versions:

```shell script
ln -s /home/tomcat/apache-tomcat-9.0.36 /home/tomcat/tomcat-latest
```

We now want to create a systemd process to run tomcat.  Create the service config file:

```shell script
sudo touch /etc/systemd/system/tomcat.service
```

and populate it with:

```
[Unit]
Description=Tomcat 9 servlet container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/jre"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"

Environment="CATALINA_BASE=/home/tomcat/tomcat-latest"
Environment="CATALINA_HOME=/home/tomcat/tomcat-latest"
Environment="CATALINA_PID=/home/tomcat/tomcat-latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/home/tomcat/tomcat-latest/bin/startup.sh
ExecStop=/home/tomcat/tomcat-latest/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

Reload the daemon and start and enable the service:

```shell script
sudo systemctl daemon-reload
sudo systemctl start tomcat.service
sudo systemctl enable tomcat.service
```

##  Install GeoServer
We follow the instructions on the 
[web archive installation page](https://docs.geoserver.org/latest/en/user/installation/war.html).

We already have the JDK installed.  Go to the [GeoServer download page](http://geoserver.org/download/), click on 
`stable` and copy the link for the `Web Archive` package.  Back in the command line, whilst logged in as the `tomcat` 
user, get the package and unzip:

```shell script
wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.17.1/geoserver-2.17.1-war.zip
unzip ~/geoserver-2.17.1-war.zip -d geoserver_2_17
```

which will unzip it in a new directory called `geoserver_2_17`.  To deploy GeoServer, either copy the file 
`geoserver.war` from the unzipped directory to `/home/tomcat/apache-tomcat-9.0.36/webapps` or create a symbolic link 
to it:

```shell script
ln -s /home/tomcat/geoserver_2_17/geoserver.war /home/tomcat/apache-tomcat-9.0.36/webapps
```

and restart tomcat. 

#### Change Data Directory

To ease installing new versions, it is recommended to change the data directory.  To do this, add the following context
parameter

```xml
<context-param>
    <param-name>GEOSERVER_DATA_DIR</param-name>
    <param-value>/path/to/geoserver_data</param-value>
</context-param>
```

to the GeoServer `web.xml` file.  This `geoserver_data` directory should be owned by the `tomcat` user. With the above 
installation instructions the `web.xml` file is located at:
  
```shell script
/home/tomcat/apache-tomcat-9.0.36/webapps/geoserver/WEB-INF/web.xml
```

Restart tomcat when finished.


## Enable reverse proxy from apache httpd

GeoServer should now be running on port 8080.  However, to access the web administration interface, we need to proxy 
certain web request to this port. This can be done either by adding a new [virtual host](webmisc.md#apache-httpd-set-up) 
or adding the following to an existing virtual host:

```
<VirtualHost *:80>

ServerName data.satsense.com
ServerAlias data.satsense.com

ProxyPass /geoserver http://localhost:8080/geoserver
ProxyPassReverse /geoserver http://localhost:8080/geoserver

</VirtualHost>
```

and restart the httpd server

```shell script
sudo systemctl restart httpd.service
```

It's possible that you'll need to set tell SELinux to allow httpd proxies:

```shell script
sudo setsebool -P httpd_can_network_connect 1
```

Note that the -P flag means that this settings persists upon restart.

Check that you can access the admin page by navigating to `http://data.satsense.com/geoserver/` or similar for your
domain.


## GeoServer configuration

### Security

If you've followed the above instructions you should be on the web admin page. You can log in with the default details:

```
username: admin
password: geoserver
```

The home page will warn you about:

1. Read the file security/masterpw.info in the GeoServer data directory and then delete it.
2. The default user/group service should use digest password encoding. 
3. The administrator password for this server has not been changed from the default.

It should be clear what to do for the last point.  For the second point, you can edit the default user/group service by 
clicking on `User, Groups, Roles` under `Security`, and then clicking `default`.

For the first point, navigate to the file `security/masterpw.info` in the [data directory](#change-data-directory) and
read its contents. It provides a master password, and should test that you can log in with that password and the 
username `root`.  You may need to enable root sign in by editing 

```
security/masterpw/default/config.xml
```

in the data directory.  Change the value in the `loginEnabled` tag to `true`, and then restart tomcat.  Remember to
change the `loginEnabled` back and to delete the `masterpw.info` file after you've checked the password works.


### Add domain to GeoServer csrf whitelist

As a layer of protection, GeoServer uses 
[csrf protection](https://docs.geoserver.org/latest/en/user/security/webadmin/csrf.html) and this can be problematic
behind a proxy.  Add the following:

```xml
<context-param>
  <param-name>GEOSERVER_CSRF_WHITELIST</param-name>
  <param-value>satsense.com</param-value>
</context-param>

```

to the GeoServer `web.xml` file.  And then restart tomcat.

### Change Proxy Base Url

Once you've added a layer, you may find that layer preview urls are incorrect the urls may be generated with 
`localhost:8080` in them.  Clearly this isn't going to work, unless you are running GeoServer locally.

To fix this, click on `Global` under the `Settings` section and set the Proxy Base URL to

```
http://data.satsense.com/geoserver/
```

# Using GeoServer

## Add a geotiff layer

### Preparing a geotiff

We've found that compressing with predictor 2 (i.e. with `gdal_translate -co "PREDICTOR=2"`) can be problematic.  See
this [thread](https://sourceforge.net/p/geoserver/mailman/message/35858464/). 

Unfortunately, some of our code (at time of writing) outputs geotiffs with predictor 2, so you may need to change the 
predictor before inserting into GeoServer.  This can be done via `gdal_translate`:

```shell script
gdal_translate -co "TILED=YES" -co "BLOCKXSIZE=512" -co "BLOCKYSIZE=512"  -co "COMPRESS=LZW" -co "BIGTIFF=YES" -co "PREDICTOR=1" <input file> <output file>
```

Notice that the geotiff is also tiled - this is recommended, otherwise the geotiff is blocked in strips and small reads
can be very expensive.  

We also need to add overviews to the geotiff.  The current tileserver uses different geotiffs for its overviews.  
GeoServer can handle internal overviews which are generated via:

```shell script
gdaladdo -r average --config COMPRESS_OVERVIEW LZW <geotiff needing overviews> 2 4 8 16
```

The geotiff can then be put into a directory onto the server, which GeoServer can then access. The directory should be
owned by the `tomcat` user.  The live server geotiffs are contained in the folder `/home/tomcat/geoserver_geotiffs` 
which is symbolic linked to `/data/geoserver_geotiffs`. 


### Add custom style

Click `Styles` under `Data` and then `Add a new style`. Give it a name and a workspace. Leave `Format` as `SLD` and add

```xml
<?xml version="1.0" encoding="UTF-8"?>
<StyledLayerDescriptor xmlns="http://www.opengis.net/sld" xmlns:ogc="http://www.opengis.net/ogc" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.opengis.net/sld
http://schemas.opengis.net/sld/1.0.0/StyledLayerDescriptor.xsd" version="1.0.0">
  <NamedLayer>
    <Name>SatSenseRaster</Name>
    <UserStyle>
      <Title>SatSense Velocity Raster</Title>
      <FeatureTypeStyle>
        <Rule>
          <RasterSymbolizer>
		  	<Opacity>0.85</Opacity>
            <ColorMap>
              <ColorMapEntry opacity="0" color="#d7191c" quantity="-9999"/>
              <ColorMapEntry opacity="0.85" label="-7.0" color="#d7191c" quantity="-7.0"/>
              <ColorMapEntry opacity="0.85" label="-3.5" color="#eb8c6d" quantity="-3.5"/>
              <ColorMapEntry opacity="0.85" label="0.0" color="#fffffe" quantity="0.0"/>
              <ColorMapEntry opacity="0.85" label="3.5" color="#95c1bd" quantity="3.5"/>
              <ColorMapEntry opacity="0.85" label="7.0" color="#2b83ba" quantity="7.0"/>
            </ColorMap>
		  </RasterSymbolizer>
        </Rule>
      </FeatureTypeStyle>
    </UserStyle>
  </NamedLayer>
</StyledLayerDescriptor>
```

to the text box at the bottom.

### Add the layer to GeoServer

First add a workspace, which should be seen as a container for logically similar layers. Click `Workspaces` under `Data` 
and then `Add a new workspace`. Populate `Name` and the `Namespace URI`. Specifying what these mean is a TODO.

Then add a store.  Click `Stores` under `Data` and `Add a new store`.  Click `Geotiff`.  Choose a workspace, and a name
for your store. Then click `Browse` and navigate to the geotiff.

Then add a layer.  Click `Layers` under `Data` and `Add a new layer`.  Select the `Store` you've just added.  Click
`Publish`. Give it a name and a title.  Click publishing tab, and select the style to be the raster style you created.

### Protect end point

We can protect end points on a per workspace or per layer basis.  For example, in a workspace, selecting the `security`
tab, we can select which roles can access the workspace.  Adding ROLE_AUTHENTICATED means any user with a user name and 
password can access wms.
 
Other options (details is a TODO):
* Remove anonymous authentication;
* Disable WCS/WFS;
