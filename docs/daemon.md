# SatDaemon (SQLAlchemy Version)

## Introduction
SatDaemon is the project responsible for automated processing of SatSense data, from the creation of SLC images to the 
ingestion of post-processed time series in to the points database. The daemon uses a processing database to determine 
which jobs can be run.

## Configuration
The daemon and processing database are partly controlled through the config file. The config file determines which
 database the entire satdaemon project uses. Database initialisation, querying and updating are all performed on the
  database specified in the config file, unless manual override is given. The default config file is 'satdaemon.cfg
  ' within the main directory of the satdaemon repository. **This default config file must be used for all routine
   processing since other SatSense projects (e.g. SatSAR, RapidSAR etc) which need to communicate with the processing
    database assume this config file provides the information they need.**
    
Within the config file, there are a number of sections. The **'PARAMETERS'** section contains information about which
 database should be used, as well as paths for where data should be processed and software found. The
  **'PROC_MACHINES'** section lists various parameters for the different processing machines available to the daemon
  , including names, resources, ports etc. Beneath these sections are sections which give estimated resource usage
   for each kind of processing job. CPU resources are the number of CPU cores used in the process, memory is the
    maximum memory used by each kind of job in gigabytes and disk resources are an estimate of how much
     disk I/O the job uses (arbitrary units out of 100).

## Processing Database
The processing database is an SQL database such as SQLite or PostgreSQL (Postgres). SQLite was used for earlier
 versions of SatDaemon, and is currently used for testing. Postgres is used for the operational processing database. 
 Communication with these databases is achieved through the Python package SQLAlchemy, which handles the different
  SQL dialects used by different SQL database versions. **Unless you know what you are doing, do not interact
   directly with the database - use functions provided within SatDaemon instead.**
   
### Schema and Structure
The database has the following tables:

Table                | Contents
---                  | ---
files                | Each row is a zipfile with information about track, orbit, date etc
bursts               | Each row is a burst with information about track, orbit, swath and geometry
files2bursts         | Secondary table for many-to-many relationship between files and bursts
frames               | Each row is a frame and contains links to files and bursts as well as information about various processing parameters for the daemon
frames2bursts        | Secondary table for many-to-many relationship between frames and bursts
frames2files         | Secondary table for many-to-many relationship between frames and files
master               | Each row is a master image for a specific frame with processing information (one per frame)
ifg                  | Each row is a secondary image for a specific frame with processing information (many per frame)
rsar_ifg             | Each row is a RapidSAR interferogram between two dates: used for coherence and unwrap jobs
master_proc_stages   | Each row is a different processing stage for all master images (reference table)
ifg_proc_stages      | Each row is a different processing stage for all secondary images (reference table)
rsar_ifg_proc_stages | Each row is a different processing stage for all RapidSAR interferograms (reference table)
daemon_tools         | One row which the daemon checks for any new input from users
pending_jobs         | Each row is a pending job the daemon has produced with information about frame, job type etc
running_jobs         | Each row is a running job the daemon has launched with information about frame, job type etc as well as progress information
failed_jobs          | Each row is a failed job the daemon has encountered with information about frame, job type etc

The table schema are all stored in the `models.py` file within the 'database' folder of the satdaemon repository
. Each of the (non-secondary) tables is defined using the SQLAlchemy object relational mapper (ORM) syntax. Links
 between the various tables are established through these models.

### Initialisation
Prior to initialisation, PostgreSQL and PostGIS need to be installed. Beware of potential issues with conflicting versions of PostgreSQL, the psql client and PostGIS on the system itself and within a conda environment. Once postgres is installed and running, a database needs to be created, connected to and the PostGIS extension enabled in it, all of which can be done using the psql client:
```sql
CREATE DATABASE database_name;
\c database_name
CREATE EXTENSION postgis;
```

The processing database is initialised using the python script `init_database.py`, which can be found within the
 'database' folder in the satdaemon repository. The database is initialised using the default config file 'satdaemon
 .cfg', but an alternative config file can be provided using the `-d <config-file>` flag.

SQLite databases can be initialised entirely using the script `init_database.py`. This creates the database file
, creates the various tables and relationships and then inserts bursts, frames and processing stage information.

Postgres databases require a Postgres instance to be running before the database can be initialised. Once the
 Postgres instance has been set up, `init_database.py` can be used in the same way.
 
Zip files can be inserted in to the database using the script `ingest_files.py`, found within the 'database
' folder of the satdaemon repository. By default, this will insert all zip files found in subdirectories of the
 `/data` directory, provided these subdirectories start with 'T' (e.g. 'T132'). Ingestion of a file automatically
  links the file to the appropriate frames and bursts in the database.

A SQL dump file (produced using `pg_dump`) can be used to restore a back-up file or fill a new database with data from an other. This can be done once the database has been initialised as above by using the `pg_restore` command.
  
### Bringing a database up to date with the state of the file system
A new (or current) database may be out of sync with the state of the file system. In order for the daemon to know
 which jobs should be launched, the database must be checked for consistency against the file system. This should
  only need to be done very infrequently since the daemon will keep the processing database up to date as it
   processes data. 

You can use `check_db_consistency.py` in the main directory of the satdaemon repository. You must provide the config
 file which contains information about the database and file system you want to check. This script then checks every
  data and processing folder on the file system, and looks for the most advanced job for each possible date. The
   database is then updated with the information for each processing date, depending on which jobs have been run for
    that date. **Note: This is not perfect - it assumes the file system is representative of the current state, that
     files are not corrupt or half processed and that the file system is the ultimate source of truth - use with
      caution!**

If you need to create a new database, and have a current database which is up-to-date, it may be better to copy over the data from the current database to the new one. This can be done using appropriate SQL commands such as `dump`.
      
### Querying and updating the database
The safest way to query or update the database is by using the functions defined in the `dbquery.py` module
. The current way to do this is to open a python command prompt and establish a connection to the database:
```python
from dbquery import Database  # get ability to connect to database
from models import *    # get the database table models
dbq = Database()  # this is the database connection
```

With the database connection established, we can query, insert, delete or update the database.
#### Queries

```python
# to issue a simple query to the database
file_id = 'AN_EXAMPLE_FILE_ID'
result = dbq.query(File, id=file_id)

# note that 'result' is a list, even when only one object is retrieved
file = result[0]

# the 'file' object has all the attributes of the files table, which can be retrieved
path_to_file = file.abspath
date = file.date
```

#### Manual inserts, updates and deletes

If satdaemon is running correctly then you should not need to issue these commands. However, it may be necessary to
 occasionally perform some manual work on the processing database.

##### Insert
```python
from datetime import date
# to insert a new object in to the database, we make a new instance of it
new_file = File(id='EXAMPLE_ID', abspath='PATH/TO/FILE', track=100, orbit_direction='A', date=date(2019,10,1
), proc_date=date(2019,10,1))

# other fields (last_updated and links to bursts and frames) are automatically filled in by SQLAlchemy
dbq.insert(new_file)
```

##### Update
```python
# to update a pre-existing entry in the database, we issue an update command

# the update dictionary tells the database how fields should be updated
update_dict = {'track': 101}
# the update command is similar to the query command, where keywords are used to specify which rows should be updated
dbq.update(File, update_dict, id='EXAMPLE_ID')
```

##### Delete
```python
# to delete an existing row, use the delete command with similar syntax to the query command
dbq.delete(File, id=file_id)
```

#### More complex queries, updates and deletes
The query, update and delete commands above work well where equality conditions are used to identify rows of interest
. However, if you wish to query using other kinds of operators (e.g. greater than, less than, like, contains etc
) then you need to use a more complex query. See the [SQLAlchemy Query API](https://docs.sqlalchemy.org/en/13/orm/query.html#sqlalchemy.orm.query) 
page for more information. To begin one of these more complex queries, it is necessary to start a `session`. The
 simpler query, insert, delete and update commands above all begin a session inside them and execute the required
  commands. To start our own session we use the `session_scope` context manager to ensure everything runs cleanly:
  ```python
with dbq.session_scope() as session:
    result = session.query(File).filter(File.date < date(2019, 1, 20)).all()
```

### Creating a new frame and inserting it into the database
The best way to create a new frame and insert it in to the database is to use a list of bursts to define the frame
. You can use QGIS to look at the global bursts shapefile, or you can use the bursts section of the processing controller part of the admin website (https://admin.satsense.com/processing_controller/bursts) and note down the bursts you want to use in your new frame
. Once you have a list of bursts, open up a connection to the database and use `make_new_frame_using_burstlist`:

```python
from dbquery import Database
dbq = Database()

burstlist = ['132_8603_IW1','132_8612_IW2','132_8594_IW3','132_8575_IW1','132_8584_IW2','132_8566_IW3']
frame_id = '132A_CUSTOM_FRAME'

dbq.make_new_frame_using_burstlist(burst_list, frame_id)
```

Burst IDs should be in the format `{TRACK}_{ANXID}_{SUBSWATH}`, e.g. `132_8603_IW1`. The defined frame name should
 be in the format `{TRACK}{A/D}_{TEXT}_{TEXT}` e.g. `132A_CUSTOM_FRAME` or `081D_CUSTOM_FRAME`.
 
 You can check whether the new frame looks correct by plotting all the frames in the database using the python script
  `make_frames_shapefile_db.py`. The frame should also appear on the admin website.

### Setting a frame master date
Creating a new frame does not automatically give a list of all dates available for that frame. A list of dates for the frame can be obtained by using the commands below:
```python
from dbquery import Database
dbq = Database()
dbq.fill_files_table(frame_id)

# Running the command below will result in a master for the frame being automatically chosen if no master is present
dbq.update_procdates(frame_id)
# Alternatively, give a master date explicitly 
dbq.update_procdates(frame_id, user_master_date=master_date)

# If there is already a master defined and you wish to change it, use the following command.
# This will warn you if changing the master would mean reprocessing data
dbq.set_master_for_frame(frame_id, master_date)
```

## Daemon

### Overview
The daemon refers to the automated processor used by SatSense. The daemon uses a config file to determine certain
 parameters, but once launched, is largely left alone. The daemon consists of three main objects: ProcDaemon
 , ProcMachine and ProcJob objects. In simple terms, a ProcDaemon object is the daemon itself - it knows what is
  going on and communicates with the database. A ProcDaemon object knows how many ProcMachine objects it has at its
   disposal. ProcMachine objects are the processing machines themselves and know what is running on them, as well as
    their available resources. A ProcDaemon also knows what ProcJobs it can run. ProcJobs run on ProcMachines and
     these objects know all the relevant information for the job they are associated with.

### Launching the daemon
The daemon can be launched from the command line using the command `satdaemon.py`. Before launching the daemon, it is
 **very important** that you check the config file `satdaemon.cfg` is set up as you want, and that it is pointing to
  the right database! You should always launch the daemon in a Linux 'screen' so that it can continue running when you log out of the remote machines. You can run `screen -S daemon` to establish a daemon screen if one does not already exist.
  
Once the daemon is launched, it will use the specified database to check for new jobs which can be run and launch them in an appropriate order. Jobs will only be run for frames which are set to be processed. Frames can be set to process to different stages depending on need/resource availability. Frames can also be given different priorities, where higher priority frames will be processed before lower priority frames. Note that if a higher priority job cannot be launched because necessary pre-requisites have not been met, or resources are not available, a lower priority job which can be run will be launched.
  
### Controlling the daemon
Before launching the daemon, you may want to change the processing parameters for a frame. This can be achieved using the SatDaemonController object and associated methods. Various common operations can be performed using the SatDaemonController, ensuring an easy, safe and clean change to the database.
```python
from satdaemon_control import SatDaemonController
from datetime import date
sdc = SatDaemonController()

frame_id = 'EXAMPLE_FRAME'

# Turn on full processing for a frame
sdc.set_frame_full_proc(frame_id, True)

# turn frame off for all processing stages
sdc.set_frame_full_proc(frame_id, False)

# turn frame off for all processing stages
sdc.set_frame_full_proc(frame_id, False) 

# turn frame on, but only til download (all other stages turned off)
sdc.set_frame_download(frame_id, True) 

# turn frame on, but only til end of SatSAR processing (all other stages turned off)
sdc.set_frame_satsar(frame_id, True)

# turn frame on, but only til unwrap (all other stages turned off)
sdc.set_frame_process_til_timeseries(frame_id, True)

# turn frame on, but only initial processing (all updating turned off)
sdc.set_frame_noupdate(frame_id, True)

# set the processing range for the frame
sdc.set_frame_proc_range(frame_id, date(2019, 1, 1), date(2020, 1, 1))

# Set frame priority relative to other frames
sdc.set_frame_priority(frame_id, 10)
```

The daemon can also be controlled whilst the daemon is running using these commands. Each of these commands also issues a trigger to the daemon that it should recreate the job list, which it will do in light of any changes made to frame processing parameters.
  
### Turning off the daemon
The daemon should be turned off using the SatDaemonController.
```python
from satdaemon_control import SatDaemonController
sdc = SatDaemonController()

sdc.power_down_daemon()
```

This will cancel all pending jobs, but the daemon will keep running until currently running jobs are complete. The
 daemon will then stop running.
 
If you cannot use the method above, you can issue a 'KeyboardInterrupt' if the daemon is running in a terminal, or
 kill it using the PID. However, these may result in the database and/or files ending up in a confused state. This
  should  be avoided where possible.

Before relaunching the daemon, it is important to check the database is not in an incorrect state. Prior to relaunching the daemon, there should be no jobs in the 'running_jobs' table, and there should be no rows in the ifg/master/rsar_ifg tables with a 'proc_status' of 1 (running). A daemon crash may not cause previously launched jobs to crash, and the best way of checking whether there are still jobs running is to check for running python processes on each of the processing machines, which can be done by running the command below in the shell:
```bash
ps aux | grep python
```
Jobs can take hours to finish. Before relaunching the daemon, let all jobs which had been launched by the daemon finish.
  
### Error reporting
Errors may occur in the daemon itself (i.e. within `satdaemon.py`) or in one of the jobs launched by the daemon
. Errors in the daemon itself will cause the daemon to crash. These errors will be reported to the terminal and to a
 dedicated Microsoft Teams channel. Any jobs which are marked as running in the processing database will be reset so they can
  be rerun when the daemon is restarted.
  
Errors within launched jobs are handled by the JobLogger object, created by the launched job itself. Each launched job 
communicates with the database and Micorsoft Teams using the JobLogger, reporting the processing status of the job, any errors
 or warnings and, in some cases, the progress of the job. Sometimes, errors within launched jobs are not caught by
  the JobLogger, and the job stops unexpectedly. In these cases, the return code of the job should be caught by the
   daemon and the daemon reports to the database and Micorsoft Teams that an error has occurred.

## Handling Crashes
Jobs can sometimes crash unexpectedly, and this can mean the database does not get correctly updated. Possible problems which may occur are:
1. The master/ifg/rsar_ifg row in the corresponding table is not correctly updated. The 'proc_status' column may be stuck on a value of 1 (running), or it may correctly be changed to 4 (failure).
2. The 'running_jobs' table may not have the job removed. This means the job will never be launched again as the database states it is already running. This row needs to be deleted in order to relaunch the job.
3. An error can occur in the actual `satdaemon.py` script, which causes the daemon itself to crash. A daemon crash means the daemon stops running and no more jobs are launched. Previously launched jobs may still be running - make sure these jobs finish before relaunching the daemon. The best way of checking whether there are still jobs running is to check for running python processes on each of the processing machines, which can be done by running the command below in the shell:
```bash
ps aux | grep python
```
Jobs can take hours to finish. Before relaunching the daemon, let all jobs which had been launched by the daemon finish.

## Graphical Process Monitoring
We are moving towards a web-based, graphical user interface for monitoring and controlling the automated processing. At the moment, we have a web-based monitoring tool, which allows users to see which jobs are running, monitor resource usage and visualise information from the database. This can all be accessed at the admin site: https://admin.satsense.com/processing_controller

The processing controller area has pages for each frame and image as well as summaries of currently running jobs, maps of procesing progress and job failures. There is also a separate area with a map showing all global bursts, allowing for easier determination of which bursts to include in a new frame.

There is currently no ability to change the daemon or database from the processing controller pages. The processing controller is currently only set up in 'read only' mode.
