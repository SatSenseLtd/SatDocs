# "What Mike does"

This document contains a list of ad hoc jobs that that need to be taken over.  It also contains high level instructions for each job along with links to other relevant documentation, if applicable.

# Contents

- [Ingest new frame into portal](#ingest-new-data-into-portal) - just refers to ingesting data process and not creating a new portal from scratch.
- [Regenerate countrywide srisks for API](#regenerate-countrywide-srisks-for-api)
- [Regenerate tiles](#regenerate-tiles) - tiles aka tile overviews, portal geotiffs
- [Create new portal](#create-new-portal) - just release a new portal, this assumes InSAR data already exists.
- [Add user to staging portal, portal2 or portal3](#add-user-to-staging-portal-portal2-or-portal3)
- [Generate monthly figures for API](#generate-monthly-figures-for-api)
- [Keep an eye on SLA for API](#keep-an-eye-on-sla-for-api)
- [Test against API before/after release](#test-against-api-beforeafter-release)
- [Insert new dates into live database](#insert-new-dates-into-live-database)
- [Keep an eye on SSD storage on API Server](#keep-an-eye-on-ssd-storage-on-api-server)
- [Be on hand if server/postgres bounce required](#be-on-hand-if-server-restart-or-database-bounce-required)
- [Update warm back up insar data]()
- [Update warm back up user data]()

# Ingest new frame into portal

Frequency:

- Every 1-2 months/when prompted

Actions:

- Create new database (for csk data, historically we've called it `london_csk_{today's date}`, e.g. `london_csk_20200803`). Just do this by logging into psql using `postgres` user and running `create database london_csk_...`.
- Run `insar_data_db` alembic migrations on this new database. We keep a checked out version of `satsense_domain` on the api server at `/home/centos/Projects/alembic/satsense_domain`.  There is a virtual environment in this folder too called `venv_alembic`.  Don't forget to check `alembic.ini` in `satsense_db_migrations/insar_data_db`.  To run the migrations, check current schema is empty using `alembic current` and then upgrade to most recent schema using `alembic upgrade head`.
- Give correct permissions to users for the database you've created.  For csk datasets, I've been giving `staging_pgi` full read, write and delete permissions and `satsense_ro` read permissions.  More info about setting up permissions here: https://satsenseltd.gitlab.io/Documentation/client-facing-applications/database/#2-user-creation-and-permissions
- [Use satpgi](satpgi.md#running-live-instance) to do an initial ingest into the database. Check the config before you run anything.
- [Use satdom scripts](satdom-scripts.md#running-live-instance) to generate the tiles from this database.  Again check config.  Can use `satsense_ro` user here.
- Move tiles to sensible location.  Look in `/data/tile_geotiffs` (specifically for csk look in `/data/tile_geotiffs/staging/vels_hr_asc`).  Consider doing both rural/high res or ascending/descending.
- Change satshop (or portal3 for csk data) config to use new database and satshop tileserver (or portal3 tileserver for csk data) to use new database and tiles.  Restart portal3 and run httpd reload (reload is a soft restart).  Locations of configuration files are the docs for the [portal](portal.md#live-server-locations) and the [tileservers](tileservers.md#live-server-locations).
- Check it's rendering something and check it seems reasonable.

# Regenerate countrywide srisks for api

Frequency:

- Every 1-2 months/when prompted

Actions:

- Use the script `generate_srisks_tifs.py` from the [satdom scripts](satdom-scripts.md) to regenerate the entire srisks for the api.
- Once generated, we will alter the config of the staging api (at `/var/www/api-staging/app_home/config.in` - given for convenience but could have been found in the [api docs](api.md)).  In this staging api config, change the value `all_srisk_geotiff_path` under the section `srisk` to the full path of the new srisk file.  The current value is probably somewhere on the ssd (i.e. starting with `/data`), but it is fine to test the new srisk file from the rust disks - it just might be a little slower. You may be wondering what that option `precalculated_directory_path` does - this was used when we were just serving "spatial srisks" from four different files - we now serve all srisks from one file.
- Restart the staging api to load the new config.  i.e. run `sudo systemctl stop apistaging` followed by `sudo systemctl start apistaging`.
- Run the [qa tests](https://gitlab.com/SatSenseLtd/satsense-api/-/tree/master/tests/qa_tests) against the live api (which uses the old srisk geotiff) and the staging api (which uses the new srisk geotiff due to the previous steps).  You might want to keep an eye on response times during pinging the live api, or run outside of hours.
- Compare the results from the previous step.  Would usually do this by comparing the proportions of red/amber/green or look at which polygons have changed status.
- If there is no big change then you can probably roll out the release without too much hassle.  Make groundsure aware (via list of emails I forwarded) that you plan to update some data and offer them the chance to test it against the "test api" i.e. `api.test.satsense.com`. 
- If there is a big change in the results, best to consult Karsten with what to do.  May want to highlight change to groundsure, again liaise with Karsten about who best to approach at groundsure regarding this.
- Once release is scheduled, update the test and live configs and restart each app.  The config paths/how to restart each app can be found in the [api docs](api.md). The restart of each app might need to happen on different days if groundsure want to test against the test api.

# Regenerate tiles

TODO

# Create new portal

TODO

# Add user to staging portal, portal2 or portal3

TODO

# Keep an eye on SLA for API

TODO

# Test against API before/after release

TODO

# Insert new dates into live database

Frequency:

- As often as possible

Actions:

- This refers to running the [satpgi insert-dates script](satpgi.md#updating-data-temporally).  Before running this, have a look what other satpgi are happening - check `satpgi` screens on each vm.  Tend to only run two satpgi scripts at the same time (one for each vm).  
- Wait until after 5pm on a weekday or anytime at weekend.
- Find a suitable frame to update.  We manually keep a list of how up-to-date frames are in the database. Find that [here](https://satsense2-my.sharepoint.com/:x:/g/personal/michael_west_satsense_com/EagSqosh2KlAr_-_Fni5lHIBl0XTt4Dlg9ZIY9lAfEyFZA?e=b4hvSc).  Also sometimes good to check if there is any current processing on the frame in question.  To do that, replace `<frame id>` with a frame id in the following url:  
  
  ```
  https://admin.satsense.com/processing_controller/frame/<frame id>
  ```
  
  and see if there are any `Running Dates` on that page.
- Can run satpgi insert dates as detailed [here](satpgi.md#running-live-instance).  Always check config before running to make sure you're inserting dates into the right place!
- For convenience, an example insert-date script: 
  
  ```
  python main.py insert-dates -i 103A_03733_131113 --hr-file /rust2/workspace/frames/103A_03733_131113/RapidSAR/103A_03733_131113-vel.h5 --rural-file /rust2/workspace/frames/103A_03733_131113/RapidSAR/103A_03733_131113-rural-vel.h5
  ```

# Keep an eye on SSD storage on API Server

Frequency:

- Every so often, more frequently if higher than 85% or so.  

Actions:

- Log onto api server and run `df -h` and look at `/data` usage.
- The biggest consumer of data is [inserting new dates into the database](#insert-new-dates-into-live-database), but even this doesn't eat too much database too quickly.  Really start to worry if less than 150-200GB remaining - then probably worth expanding.
- Before expanding, delete unnecessary databases/files then if still required talk to Matt (it is expensive) and if he's happy give Seth at unipart an email.
- The expansion will require a [server restart](#be-on-hand-if-server-restart-or-database-bounce-required).

# Be on hand if server restart or database bounce required

Frequency:

- When required

Actions:

- If planned and it affects API, best to let groundsure know, even if maintenance would be out of hours.
- "Server restart" usually goes completely fine and services don't need looking at.
- "Database bounce" can affect some of the application transaction handling and those applications need restarting to rectify.  We list a table of services, how you check they are fine and how to restart if they're not.


| Service Name | How to check it's fine | How to restart service| 
|--------------|------------------------|-----------------------|
| satshop      | Log into satshop.satsense.com and retrieve a time series by clicking point | `systemctl stop satshop` <br> `systemctl start satshop` |
| staging portal      | Log into portal.staging.satsense.com and retrieve a time series by clicking point | `systemctl stop portalstaging` <br> `systemctl start portalstaging` |
| portal2      | Log into portal2.satsense.com and retrieve a time series by clicking point | `systemctl stop portal2` <br> `systemctl start portal2` |
| portal3      | Log into portal3.satsense.com and retrieve a time series by clicking point | `systemctl stop portal3` <br> `systemctl start portal3` |
| admin      | Log into admin.satsense.com | `systemctl stop adminlive` <br> `systemctl start adminlive` |
| adminstaging      | Log into admin.staging.satsense.com | `systemctl stop adminstaging` <br> `systemctl start adminstaging` |
| admin2      | Log into portal2.satsense.com/admin | `systemctl stop admin2` <br> `systemctl start admin2` |
| admin3      | Log into portal3.satsense.com/admin | `systemctl stop admin3` <br> `systemctl start admin3` |
| All tile servers | Log into relevant portal and check tiles rendered | `systemctl reload httpd` |
| api live | Check new relic status/ping api/check logs for errors | `systemctl stop apilive` <br> `systemctl start apilive` 
| api test | Check new relic status/ping api/check logs for errors | `systemctl stop apitest` <br> `systemctl start apitest` |
| api staging | Ping api/check logs for errors | `systemctl stop apistaging` <br> `systemctl start apistaging` |
| api live celery process | Check celery worker logs located at `/var/www/api-live/celery/apilivew1-1.log` and `/var/www/api-live/celery/apilivew1-2.log` - make sure there are recent "succeeded" messages | `systemctl stop apilivecelery` <br> `systemctl start apilivecelery` |
| api test celery process | Check celery worker logs located at `/var/www/api-test/celery/apitestw1-1.log` - make sure there are recent "succeeded" messages | `systemctl stop apitestcelery` <br> `systemctl start apitestcelery` |
| api staging celery process | Check celery worker logs located at `/var/www/api-staging/celery/apistagingw1-1.log` - make sure there are recent "succeeded" messages | `systemctl stop apistagingcelery` <br> `systemctl start apistagingcelery` |
