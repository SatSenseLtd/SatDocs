# Postgres Ingester (satpgi)

The satsense postgresql ingester (aka PostgresIngester, satpgi) is a set of scripts that takes a set of hdf5 files (typically called "velocity" and "data" files) and inserts them into a postgresql database.

Instructions for how to set up satpgi are kept in the project [README](https://gitlab.com/SatSenseLtd/postgresingester/-/blob/master/README.md).  What that README does not make explicit is where everything is installed on the unipart instances.  We detail that in this documentation, along with brief instructions with what to run to keep the database up to date.

# Contents

* [Unipart Server Set Up](#unipart-server-set-up) - where files related to satpgi are kept on the unipart instances.
* [Overview of commands](#command-overview) - what commands to run and when.

# Unipart Server Set Up

The "live" instance of the satpgi is run from the unipart vm machines (vm3 and vm4).  This instance of satpgi is set up on the unipart servers inside a conda environment called `satpgi`.  The editable repositories that this environment points to are kept in the following directories:

```
/data/Software/satsense-domain
/data/Software/postgresingester
```

## Running live instance

We typically run the ingester as the `centos` user (try running `sudo su centos` if not logged in as `centos`).  Herein we assume that we're logged into either vm as `centos`. 

We usually maintain a screen running on each vm called `satpgi`.  To see if such a screen exists, run `screen -ls` and it should be listed.  If it is running, then you can connect to the screen using `screen -r satpgi`. If it isn't running then you can create a new screen via `screen -S satpgi`.  

Make sure the `satpgi` conda environment is activated using `conda activate satpgi`.  It is then best to run `satpgi` from the `/data/Software/postgresingester/ingester`, and you can access the help page using:

`python main.py --help`

Don't forget to update `config.ini` with any values.  Do check the following:

- `db_name` - the database name (very important)
- `db_username`
- `db_password`
- `include_frame_boundary` - this should be `True`, but the data might not be available if it is not sentinel data (e.g. India or csk data).  This will insert a polygon representing the boundary of the frame, these are contained in a file called `ingester/UK_frame_coords_ASF.txt` .  Polygon boundary file name can be changed in config, see `asf_frame_coords_name` in `ingester/config.ini`.

## Updating live instance

It's probably best if you don't do this step if there are any running commands. Check for any screens called something like `satpgi`.

Otherwise, it's just a matter of git pulling the latest repositories in:

```
/data/Software/satsense-domain
/data/Software/postgresingester
```

## Logs

Logs are contained in the directoy `/data/Software/logs` (although the reader could have checked `config.ini` to find that out). We generally only run one instance per vm, and we append the vm name onto the log file to keep these log files distinct.

## Other instances of satpgi on unipart machines

There is another conda environment called `test_satpgi`, this should only be used on vm4.  The corresponding installed editable repositories are contained at:

```
/home/centos/satsense-domain
/home/centos/postgresingester
```

on vm4.  This instance is useful for testing new datasets or ingesting into different databases.


# Overview of commands

At the time of writing, printing the help page for satpgi will yield information about the following commands:

```
insert-csv          Insert HDF5 files using metadata stored in a csv file.
                    This is particularly useful for setting up more than
                    one file for ingestion at once. See example_input.csv.
initial-insert      Insert using flags. This is limited to only ingesting
                    one HDF5 at a time, but may be more useful for
                    automation.
reingest-points-for-frame
                    Reingest the points for frame. This will read the
                    dates that are currently in the database and only
                    ingest those dates. Points not in the database will be
                    inserted, points that shouldn't be in the database
                    will be deleted, and all points that are already in
                    the database will be fully updated.
insert-dates        Update points in the db with dates that aren't in the
                    database already.
delete-dates-in-frame
                    Remove dates in the time series for points within
                    frame.
update-dates        Update dates in the db that already have a placeholder
                    in the database. Option to start from specific batch.
                    Useful if an 'insert-dates' script had to be paused.
update-fields       Update fields on points. There must be exactly one of
                    this type of data per point (for example, velocities
                    and not displacements)
delete-frame        Delete both high resolution and rural points from the
                    db for a given frame.
delete-point-ids    Delete points in database with ids defined in a csv
                    file.
```

Whilst these are documented, we add some more context about these commands i.e. some of the circumstances behind why we might use each command.  Roughly these are split up into the categories

* Inserting a new frame
* Updating data spatially - adding, updating or removing points.
* Updating data temporally - updating the time series for points.
* Updating fields - updating any data on points that isn't the time series.

## Inserting a new frame

Imagine you have an empty database, or a brand-new processing frame to play with - these scripts are what you want.  We initially wrote the command `initial-insert`, but this can be laborious if we want to run loads of insertion scripts in a row. To solve this problem, we wrote `insert-csv`.  In most cases, these two commands are essentially equivalent. `initial-insert` former offers slightly more options, for example, the ability to filter by extent upon ingestion - this might useful for ingesting into a demo portal database.

These commands are handle points in batches and should be fine to run at any point without too much other load on the database (i.e. not too many satpgi scripts at once).

## Updating data spatially

If a frame has a number of suspicious points, then we allow a few mechanisms to handle this:

* `delete-frame` - nuclear option, it will delete all points for a frame.  This probably shouldn't be run.
* `delete-point-ids` - Whilst it is easy to run a sql statement to delete point ids, historically it has been preferred to run scripts that are tested.  This takes in a csv of ids and will delete points in the database with those ids.
* `reingest-points-for-frame` - the outcome of this is equivalent to running `delete-frame` and then `initial-insert` except there is never a point where the database doesn't contain points for the frame.  It ensures a clear reingest if the Reliable Pixels dataset in the velocity file has been updated.  Done in batches and should be fine to run at any point without too much other load on the database.

## Updating data temporally

For frames whose time series has been updated in hdf5 files.  For instance, if there is a new (or set of new) date(s) in the time series then you would run `insert-dates`.  If you find some processing error for one date in a frame then you would run:

* `update-dates` if the error has been fixed in the hdf5 files and you need to get this data into the database.
* `delete-dates-in-frame` if the error has not been fixed in the hdf5 files - perhaps it will take some time to figure out the issue.

The script `update-dates` handles points in batches and should be fine to run at any point without too much other load on the database.

The script `delete-dates-in-frame` is quite write intensive - it has to rewrite every time series for every point in one transaction (although that transaction is long-lived).  Best to start this in the evening or weekend.  Wouldn't have more than one of this kind of script running at once.

The script `insert-dates` is the most commonly used script. It consists of two parts:

* Adding NaNs as placeholders in every points time series
* Updating those points with real values

The first part is so to ensure that the database is always in a consistent state.  It is quite write intensive, and best to start in the evening or weekend - depending on the frame it can take a long time.  The second part is handled in batches so should be fine to run alongside other scripts.  Indeed, once this script has started its second phase, then another instance of this script can be run.

# Updating fields on points

There is a generic script called `update-fields`.  This script can handle updating fields that are represented by a single (non-array like) column in the database.  A full list of the possible fields to update is given in the help page for this script (i.e. run `python main.py update-fields --help`).  Common use cases include and update of the velocities for a frame or if a new column/field to the database has been added then this script can easily be amended to allow updating.

This is handled in batches and should be fine to run at any point without too much other load on the database.
