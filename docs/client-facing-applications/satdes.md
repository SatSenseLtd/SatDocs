# Satdes (Data Extraction Service)

This is a few scripts that are used to make extract data from an InSAR database.  

Instructions for how to install an instance of satdes are kept in the project [README](https://gitlab.com/SatSenseLtd/data-extract-service/-/blob/master/README.md).  What that README does not make explicit is where we typically run these scripts on unipart servers.  We detail that in this documentation.

We also have documentation for users of satdes. Part of this is held the help pages when one runs `satdes --help`. We also have a useful document [here](https://satsense2.sharepoint.com/:w:/s/SatSense/EUrb0rCGUkhFo8D-qOQHVZMBNU_QCN-Os4WBD0nN0rktzg?e=slkT3J). 

# Contents

* [Unipart Server Set Up](#unipart-server-set-up) - where files related to the satdes scripts are kept on the unipart instances.
* [Enable Another User Access to Satdes](#enable-another-user-access-to-satdes)

# Unipart Server Set Up

The "live" instance of the satdes scripts should be run whilst logged into the api server.  This instance of satdes is set up inside a virtual environment which is held:

```
/rust2/workspace/satdes/venv
```

We note the `satdes` and its dependency `satdom` are installed in this environment as a non-editable dependency. To update the above venv instance:

* Update (i.e. git pull) `satsense-domain` and `data-extract-service` repositories located in `/rust2/workspace/satdes/live`.
* Activate the above venv `source /rust2/workspace/satdes/venv/bin/activate`
* Uninstall `satdom` and `satdes` by running `pip uninstall satdom satdes`.
* Check whether dependencies need updating, and is so, run `pip install -r requirements/production.txt` from root of `data-extract-service` repository.
* Reinstall new versions of `satdom` and `data-extract-service`, that is run `pip install .` from `satsense-domain` root and `data-extract-service` root.

One can find the `config.ini` at `/rust2/workspace/satdes/live/data-extract-service/satdes` and logs for this application are held in the folder `/data/logs/satdes`.

# Enable Another User Access to Satdes

As centos on the api server, create a new user

```
sudo adduser <new username>
```

Then log in as that user (e.g. run `sudo su <new username>`) and create authorized keys:

```commandline
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Then add a public ssh key for that user to authorized keys. The user should be able to use their ssh to log in to the api server. Once in they can activate satdes via `source /rust2/workspace/satdes/venv/bin/activate` and run `satdes --help` to access the help page.

It might then be useful for a user to have a space to extract files to.  For example, historically we have set up directories for users inside `/rust2/workspace/data_extract`.  For example you might create a new folder:

```
mkdir /rust2/workspace/data_extract/<new username>
```

and then set the permissions:

```
chown <new user>:centos /rust2/workspace/data_extract/<new username>
```
