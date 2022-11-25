# Setting up AWS for processing

## Starting EC2 instance
EC2 instances can be started from the AWS EC2 console. If just a single instance is required, simply select the stopped instance in the Instance screen, and select Actions -> Instance Settings -> Modify Instance Type to select the instance type required. Note that not all instance types are available, t3 (general purpose) and r5 (higher memory) instances work. A t3.medium instance works well for SatSAR, r5.xlarge works well for RapidSAR and beyond. Note that starting an instance like this does not come with additional storage attached, see the section on Setting up EBS volumes below. All software should be installed and environments set up for basic processing, except for the database, see the section on RDS database processing below. It's probably a good idea to run and apt upgrade and pull changes in from gitlab on the various bits of software. 

If multiple instances are required, we can create an image of the basic stopped instance. If necessary, start the instance and ensure the latest updates have been applied (apt and gitlab). Then from the console, go to Actions -> Image and templates -> Create image. Give the image a name and click "Create image". From the main console window, the image is now available from the AMI screen. This image can now be used to launch images. Ensure that the vpc is (vpc-770a581f) and that the subnet mask is the default for eu-west-2a. On the next screen, storage can be defined, or this can be added at a later time, see section on Setting up EBS volumes.

One the instance is running (this can take a minute or more), you can ssh into the EC2 instance using the IP address or URL of the instance, which you can find on the Instances screen. The username is ubuntu. From the instances screen, you can also stop the instance, or run the shutdown command from the ssh session terminal. Note that storage continues to be billed, even if the instance is stopped. Any unused instance (besides the basic instance), including their EBS volumes, should be deleted.

## RDS database setup
As our processing database is currently not accessible from outside Unipart network, we'll have to set up a specific database for AWS processing. Luckily this is a fairly streamlined process consisting of two parts, commissioning the database from the AWS RDS console, and setting up the specific requirements for the SatSense processing database. 

### RDS console
From the console, we can determine the type of database we want, and set up security and connectivity settings. The steps listed below will set up a database suitable for SatSense processing:

- From the main dashboard, click the Create Database button, and select "standard create" and "PostgreSQL". For the version, go with the latest subversion of version 12, at the time of writing version 12.8-R1.

- Give the database and identifier. Leave the username, and choose a password (and remember it!).

- Select burstable classes, type db.t3.micro

- Change storage type to "General Purpose SSD (gp2)" and leave allocated storage to the minimum 20 GB.

- Untick "Enable storage autoscaling" and select "Do not create a standby instance"

- Select the default VPC and subnet groups (vpc-770a581f), set availability zone to "eu-west-2a" and enable public access



After this, create database and wait for it to come online. This might take 15-30 minutes.

### Additional database setup
Now we switch to a Linux terminal on our local system or an EC2 instance. EC2 instance is probably a better option, but not mandatory. If using a local terminal, ensure a postgresql client version 12 is installed. If using EC2, ensure it is a t2/t3.medium or above, as some memory will be required for the ingestion of bursts.

- Set the $DBIP environment variable to the correct enpoint url. After the database has initialised, you can find the url in the Databases screen, under "Connectivity and Security". Here you can also find the port number, and ensure that the VPC and security groups match the EC2 instance you'll use.

- Log into database from EC2 instance or local system: `psql --host=$DBIP --port=5432 --username=postgres --password`

- From psql, create satsense_live database `CREATE DATABASE satsense_live OWNER=postgres;`

- Switch to database `\c satsense_live` 

- Activate Postgis, see https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.PostGIS.html

- Log out of database

- Ensure $DBIP points to the RDS DB, and make sure the password, port and user are correct in satdaemon.cfg.

- From the satdaemon directory, run init_database.py `init_database.py -d satdaemon.cfg -f database/UK_frame_coords_ASF.txt -b database/UK_frame_burst_ids.txt -l database/burst_corners_llh.dat` This will take a while (30 mins or so?)

After this, you should be good to go. You can test the database by going into the satdaemon/database directory and running `python make_frames_shapefile_db.py`. If it completes at least one frame, the database is working as it should and you can stop the script with CTRL+c. That's it.

## Setting up EBS volumes
Disk space for EC2 instances is provided by Elastic Block Storage (EBS) volumes. EBS storage is charged by storage volume provided (not used!), and costs the same whether it is attached to a running instance, attached to a stopped instance or not attached. Except when processing, the only EBS volume should be the one that has the installation files of the stopped instance on it. 

EBS volumes can be created and deleted at will. This can be done in the EC2 console, under Elastic Block Store -> Volumes. Click the "Create Volume" button and select volume type, which normally should be general provisioned SSD. Set the size and ensure Availability zone is the same as the EC2 instance (normally eu-west-2a) and click "Create Volume". The volume should be showing as "Available" momentarily.

To use a volume inside an EC2 instance, it needs to be attached. Under normal conditions, EC2 volumes can only be attached to a single EC2 instance! Select the volume you wish to attach, and select Actions -> Attach volume. Select the EC2 instance you wish to attach the volume to. 

To be able to use the volume, it has to be formated (if it is newly created) and mounted to a mount point. Volumes have to be remounted every time the instance gets restarted, unless there is an entry in fstab to automatically mount them. This is not receommended, as we likely want to switch volumes in and out. To first need to know where the volume is attached to:

`sudo lsblk`

This will show you a list of all attached devices. The EBS volume name will likely be nvme*n1, where * is 1,2,3,etc. We can see whether the volume needs formating using:

`sudo file -s /dev/nvme*n1`

Replacing * with the corresponding number. If the volume is already formated, the output will look similar to:

`/dev/nvme1n1: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)`

If not, it will look like this:

`/dev/nvme2n1: data`

To format the volume, use:

`sudo mkfs -t xfs /dev/nvme*n1`

To mount the disk, create a suitable mount point (e.g. /data, /workspace) and use:

`sudo mount /dev/nvme1n1 /mountdir`

Ensure you have write permission in the directory the volume is mounted to, and you're good to go.

Volumes can be modified on the fly. This is useful when e.g. you need additional space. Not that volumes can only be increased, not decreased! To increase a volume, modify the volume through the Elastic Block Store screen in the EC2 console. Select the volume and go to Actions -> Modify volume. Up the size to the desired value and click "Modify". Wait for the volume to reach Optimizing stage. The volume should now be ready to be expanded from inside the instance. If the volume was mounted, you can use the following command to attach the expanded storage to the same mount point, without disruption:

`sudo xfs_growfs -d /mountdir`

More information on growing an EBS volume can be found in the AWS docs (https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)

## S3 archiving
Simple Storage Service is a bucket based storage system allowing a range of storage performance levels, ranging from high performance to infrequent access glacier storage. We mainly use S3 for their glacier storage. Syncing large volumes of data from EBS to S3 (or vice versa) is best done using aws-cli, which can be installed using apt-get and configured as follows:
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html

### Setting up a bucket
The S3 dashboard can be accessed through the AWS console. A new bucket can be created using the Create Bucket button. Give it a name (Stringent rules, e.g. no capital letters or underscores allowed) and ensure it is in the eu-west-2 region. All other settings are fine. The storage class (https://aws.amazon.com/s3/storage-classes/) is best set through a lifecycle rule. Typically, we'll want to use glacier storage for larger files. To set this up, go to the bucket and in the Management tab click Create lifecycle rule. Give the rule a name (e.g. Glacier), and specify Minimum object size (1000 or 10000 kb) to avoid small files being archived, as there is a cost per object. In the Lifecycle action, tick "Move current versions of objects between storage classes", and in the box below select "Glacier Flexible Retrieval" in the Storage Class drop-down menu. Set the "Days after object creation"value to somewhere between 3 and 5 days, and creeate the rule. This will ensure any object larger than the minimum object size will automatically get moved to Glacier storage after 3-5 days.

### Moving data to a bucket
Data can be added through the dashboard directly from a local machine, but for bulk transfers, the aws cli is best used. From an EC2 instance, this would look something like:
`aws s3 sync <Local directory of file names to sync> s3://<bucket-name>

### Retrieving data from a bucket
The data sync works in reverse as well, but only if the data is available in the Standard storage class. This means any file that is in Glacier needs to be retrieved first. To do this, select any files that need retrieval and selecting "Initiate Restore" in the "Actions" menu. Note that this can only be done on files (not whole directories), and only on files that are in Glacier, selecting a mix of Glacier and Standard files will leave the retrieval option greyed out. Set the number of days that files will remained restored, and select how quickly files should be restored. The quicker, the costlier. Then click on Initiate Restore at the bottom. Note that the files will not be listed as Standard storage class after restoration has completed, the best way I've found to see if the restore has completed is to click on the files, and ta box in blue will say whether the restore is in progress or has completed. 

To sync restored files using the AWS cli, note that the --force-glacier-transfer is required, as the file is still marked as being a glacier object:
 `aws s3 sync s3://<bucket-name>/<DIR or FILE to sync> <Local Dir to sync to> --force-glacier-transfer
  



