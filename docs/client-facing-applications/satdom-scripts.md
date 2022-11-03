# Satdom Scripts

This is a few scripts that are used to make assets for other applications.  The scripts produce derived data from an InSAR database or geotiffs.

Instructions for how to install an instance of satdom are kept in the project [README](https://gitlab.com/SatSenseLtd/satsense-domain/-/blob/master/README.md).  What that README does not make explicit is where we typically run these scripts on unipart servers.  We detail that in this documentation.

# Contents

* [Unipart Server Set Up](#unipart-server-set-up) - where files related to the satdom scripts are kept on the unipart instances.
* [Overview of each script](#command-overview) - what each script does and where the output is used.

# Unipart Server Set Up

The "live" instance of the satdom scripts is run from the unipart vm machines (vm3 and vm4).  This instance of satdom is set up on the unipart servers inside a conda environment called `satdom_scripts`.  The editable repositories that this environment points to are kept in the following directories:

```
/data/Software/satsense-domain
```

Note that this is shared with the [satpgi setup](satpgi.md#unipart-server-set-up), so updating this instance needs to be in tandem with updating satpgi (this can be changed if it proves a pain - just set up a new instance).


## Running live instance

We typically run the ingester as the `centos` user (try running `sudo su centos` if not logged in as `centos`).  Herein we assume that we're logged into either vm as `centos`. 

We usually maintain a screen running on each vm called `satdom_scripts`.  To see if such a screen exists, run `screen -ls` and it should be listed.  If it is running, then you can connect to the screen using `screen -r satdom_scripts`. If it isn't running then you can create a new screen via `screen -S satdom_scripts`.  

Make sure the `satdom_scripts` conda environment is activated using `conda activate satdom_scripts`.  It is then best to run `satdom_scripts` from the `/data/Software/satsense_domain/scripts`. You can see the help page of each script using:

`python <script file.py> --help`

Don't forget to update `config.ini` with any values on databases you need.

## Updating live instance

It's probably best if you don't do this step if there are any running commands. Check for any screens called something like `satdom_scripts` and `satpgi` (since `satpgi` uses this instance of `satdom`.

Otherwise, it's just a matter of git pulling the latest repositories in:

```
/data/Software/satsense-domain
```

# Overview of each script

At the time of writing we have the following scripts:

* generate_tiles.py
* generate_srisks_tifs.py
* filter_geotiff.py

## generate_tiles.py

This produces the overview tiles for the tileserver. When the portal gets to zoom level 14 or below there too many points to render. To solve this, we take an average of the points inside a square region (which we refer to as a tile) and store those inside a geotiff.

The script generates these tiles, it will create a geotiff of extent as per "bng_extent" in `config.ini` (which can be altered).  The `resolutions` as committed in the project are a good suggestion to use, but could be changed if the data changes.

The script generates the finest (smallest number - which refers to the distance in meters between each cell in the geotiff) resolution first using a cartesian coordinate system e.g. BNG.  It then uses gdal scripts to generate the other resolutions.  Finally, a gdal warp method is used to reproject the geotiffs into wgs 84 (epsg 4326) - the portal/tileserver expects to use spatial data in this projection.

## generate_srisk_tifs.py

This produces big srisk geotiffs, see [here](https://satsense2.sharepoint.com/:w:/s/SatSense/ER5fLpcEBdBJtrqD2Rt59nYBMjsVVNJpd4C6AFStW8UyMg?e=ZEKVQV) for more information about srisks.  These are currently used to cache data for the api because the calculation took too long for bigger query areas.  There are a couple of non-trivial options for this script:

* `--include-temporal-srisks` - this was added in for historical reasons.  Temporal srisks refer to srisks generated from the time series of the data (i.e. sriska, sriskr).  Initially, we attempted to only cache the spatial srisks (the non-temporal srisks i.e. sriskabs, srisk100, sriskp, sriskg). By offering this option, we allow the user to generate of only spatial srisks by default (but also give them the option of creating everything).
* `--store-in-single-geotiff` - after a while, we needed to improve the speed of the api and we realised that we could do this by also caching the temporal srisks.  However, we realised that this would mean opening six files. To improve on this, we just create one file and store the srisks as separate bands.

For the sake of clarity, the cached data that the api uses expects one to use both of above flags when extracting the srisks.

## filter_geotiff.py

This script will take in a geotiff and output another geotiff which is a spatially filtered (like sriskabs) version of the input geotiff.  

Groundsure provide a map layer of satsense data in their reports.  This layer is a velocity geotiff that is filtered using a spatial filter of 100m.  To generate this geotiff for groundsure, you may take the 10m resolution BNG-projected geotiff of the output of `generate_tiles.py` and run it through this script with a filter of 100m.
