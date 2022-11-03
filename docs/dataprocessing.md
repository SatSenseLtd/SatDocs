# Overview
The SatSense InSAR processing chain consists of four main steps:
1. SatSAR: Ingest SAR data from archive and generate single-master interferograms.
2. RapidSAR: Coherence estimation and unwrapping of interferograms.
3. Time-series inversion: Cumulative displacement time-series and velocity.
4. Post-processing: Multiple filtering steps to reduce noise and mitigate signals from tropospheric delays.

# SatSAR
SatSAR is an interferometric software package developed by SatSense, originally derived from [LiCSAR](https://gitlab.com/LiCS/LiCSAR/). SatSAR uses the [Gamma](https://www.gamma-rs.ch/no_cache/software.html) SAR processor as a backend to ingest SAR data from archive, and is developed predominntly in Python. The package is currently undergoing a re-organisation, so this page is subject to change.

## Package organisation
The batch processing to form interferograms consists of three main steps:
1. [make_images.py](https://gitlab.com/SatSenseLtd/SatSAR/blob/master/bin/make_images.py)
2. [coreg.py](https://gitlab.com/SatSenseLtd/SatSAR/blob/master/bin/coreg.py)
3. [make_interferograms.py](https://gitlab.com/SatSenseLtd/SatSAR/blob/master/bin/make_interferograms.py)

### Gamma functions
The four-step scripts described above act as the front end to the processor. Under the hood, they use the Gamma SAR processor: [gamma_functions](https://gitlab.com/SatSenseLtd/SatSAR/blob/master/python/gamma_functions.py) module. The function names in this module correspond one-to-one to Gamma functions, and more documentation on them can be found in the Gamma manual.

### Database integration
The Sentinel-1 satellite acquires data in bursts and swaths. To keep track of these, SatSAR can be run using input files (polygon and ziplist), or using the database. The queries into the database are contained in the [satdaemon dbquery](https://gitlab.com/SatSenseLtd/satdaemon/-/blob/master/database/dbquery.py) module.

## Processing directory structure
A typical SatSAR working directory will contain a number of directories. Below is a description of the main directories:

### DEM
The DEM directory contains the Digital Elevation Model necessary for processing. This can be created using the [mk_srtm2insar](https://gitlab.com/SatSenseLtd/SatSAR/-/blob/master/scripts/mk_srtm2insar) script. You can use whatever DEM you want, but *the files must be named as listed in the table!* Below is a table of files that should be in the DEM directory.
| File         | Description |
|--------------|-------------|
| dem_crop.dem | Binary file containing the DEM heights |
| dem_crop.dem_par | Parameter file associated with the DEM |
| dem_crop.dem.ras | Sunraster preview of the DEM |

### geo
The *geo* directory contains files related to the sensing geometry, like height of every point, latitude, longitude and look angles. Below is a list of important files in this directory.

| File     | Description      |
|----------|------------------|
| [Masterdate].hgt | Binary file containing the height of every point in the master MLI/IFG|
| [Masterdate].lat | Binary file containing the latitude of every point in the master MLI/IFG |
| [Masterdate].lon | Binary file containing the longitude of every point in the master MLI/IFG |
| [Masterdate].lt | Complex binary file containing Lookup table between the master MLI and EQA.dem |
| EQA.dem | Binary file containing a crop of the DEM fitted to the processed scene |

Most of these files have associated preview and/or parameter files. There are quite a few other files in the directory, but you're less likely to need to know about them. All of the above files are created during [coregistration](https://gitlab.com/SatSenseLtd/SatSAR/blob/master/bin/coreg.py), except for the lat and lon files, which are created during post-processing using a separate [script](https://gitlab.com/SatSenseLtd/SatSAR/-/blob/master/bin/get_lonlat.py).

### IFG
The IFG directory contains files related to interferograms. Within this directory, there are two options for the directory structure, depending on the chosen method during processing. Usually, inside the IFG directory, there will be a singlemaster directory. Inside here, there is a directory for every combinations formed, which will be one combination for each slave image. The other option is that you find combinations directly in the IFG directory. This means that the short temporal baseline combination was chosen, and the number of combinations per image depends on the maximum combination parameter that was used.

Whichever method is used, the files inside each combination directory looks the same.
| File | Description |
|------|-------------|
|[combination].diff | Wrapped, unfiltered interferogram |
|[combination].sim_unw | Estimated topographic phase |

### SLC
Single Look Complex data. This will only contain the master directory, apart from during the SLC generation.

### RSLC
Registered Single Look Comple data, contains dtaa for each image/acquisition.

_Note that there are several additional auxillary directories also created._

## Tutorial
At the end of this tutorial you will have:
1. Setup software modules including the SatSAR module
2. Learned basic SatSAR workflow
3. Used SatSAR to process a test frame

Some acronyms/terms used throughout this tutorial:
| Item        | Description                                                    |
| ----------- | -----------                                                    
| SatSAR      | The version of LiCSAR used by SatSense LTD.                    | 
| Gamma       | Commercial software used for various stages of InSAR processing.                

**Throughout the tutorial, wherever you see text between arrows:**
```shell
<LIKE_THIS>
```
**This refers to somewhere you are required to provide an input in a bash terminal, or (`python`) terminal (if speficied).**

### Obtaining your own local version of SatSAR
First, obtain an ssh key for GitLab by following the instructions here:  
https://docs.gitlab.com/ee/gitlab-basics/create-your-ssh-keys.html  
Instructions to generate your ssh key can be found here:  
https://docs.gitlab.com/ee/ssh/README.html  

Following this, go to the area you wish to set up your local version of SatSAR, for example:
```shell
cd <PATH_TO_YOUR_CLONE_LOCATION>
```
e.g.
```shell
cd /home/karsten/
```

Activate your ssh key (create with ssh-keygen if necessary):
```shell
ssh-agent $SHELL
```
```shell
ssh-add ~/.ssh/<keyname>
```
Note that you have to do this every time you want to interact with the remote repository.

Now, go to the SatSAR GitLab page (https://gitlab.com/SatSenseLtd/SatSAR) and copy the SSH URL found at the top of the page (you may need to swap to SSH from HTTPS using the drop down box). Clone the remote GitLab repository to your local system:
```shell
git clone <GITLAB_URL>
``` 
The following should work with the current SatSAR URL:
```shell
git clone git@gitlab.com:SatSenseLtd/SatSAR.git
```
We will also need the satdaemon repository, which is also on gitlab. Navigate here, and clone it from the same location as you did for SatSAR.

### Setting up SatSAR
SatSAR uses Gamma, implemented within the SatSense software written in python. In order to begin processing, you need to set up the appropriate software. At SatSense, we have Gamma and python installed as modules or applications. Begin by setting these up using:
```shell
module load canopy python-libs gamma/20180704
```
This will import various python modules and set up Gamma as required for processing. 

Next, source the LiCSAR software. There is already a LiCSAR_source.sh file in your local SatSAR folder, which should be copied to a different location and updated as follows:
```shell
LiCSARpath=<PATH_TO_YOUR_SATSAR_FOLDER>
```
e.g.
```bash
LiCSARpath=/home/karsten/SatSAR
```
Source your updated LiCSAR_source.sh file:
```shell
source <PATH_TO_YOUR_CONFIG_FILE>/LiCSAR_source.sh
```
e.g.
```shell
source /home/karsten/LiCSAR_source.sh
```
** You could put these commands in your bashrc/cshrc file such that they run when you load a new terminal **  

Finally, make an empty folder entitled "temp" and then make a new environmental variable ,$LiCSAR_temp,  in your bashrc/cshrc file, by adding the following command:
```shell
export LiCSAR_temp="<PATH_TO_YOUR_TEMP>"
 ```
e.g.
```shell
export LiCSAR_temp=/home/karsten/temp
```

### Using SatSAR on the SatSense Virtual Machines
This will take you through getting started processing on the Virtual Machines (VM). There are currently two virtual machines (VM3 and VM4) at SatSense for processing. This also requires a _.bashrc_ file with correct paths and loading the satsense_live conda environment.

To access the virtual machines from your local Ubuntu terminal:
```shell
ssh -X -p 2223 user@93.93.133.78
ssh -X -p 2222 user@93.93.133.78
```
Now you are on the virtual machine, check the data storage availability on the rust disks (rust0 to 3, 115 TB each) prior to starting processing:
```shell
df -h
```
Set the emptiest of these to store the downloaded files through the intial processing stages:
```shell
RUSTDIR=rust0
echo $RUSTDIR
```

Downloaded SAR data is stored in:
```shell
/data/
```
Any subsequent frame products are stored in:
```shell
/workspace/frames/
```
When processing, login to centos (root privelidges account):
```shell
sudo su centos
```
If using the database for frame processing (`python`):
```shell
from dbquery import Database
from dbquery import Frame
dbq = Database()
```
Several of the following steps take a while to run so do this in a `screen`:
```shell
screen -S proc_guide
```

### Directory set-up and data download

Prior to dowloading data, check if a track data directory already exists (e.g. T000, from pre-existing defined frames).
```shell
TRACK=000
ORB=D
ls /data/T${TRACK}
```
If not, create this in the data directory of the rust disk which has the most remaining space.
```shell
cd /${RUSTDIR}/data/
mkdir T${TRACK}
```
The data directory must now be linked to rust2.
```shell
cd /rust2/data/
ln -s /${RUSTDIR}/data/${TRACK} .
```

Similarly we need to create a frame processing directory for processing. Framenames are self defined but must contain the track name and orbit, and then two additional strings separated by underscores (e.g. 000A_UK_131313). The second string can be used to direct the user to the location (either name or colatitude), whilst the third string is typically used to inform a user of the number of bursts in each subswath within the frame (see ASF/SciHub/QGIS).

If you want make typing things in easier, set a bash environment variable for the framename:
```shell
FRAMENAME=${TRACK}${ORB}_UK_131313
echo $FRAMENAME
cd /${RUSTDIR}/workspace/frames/
mkdir $FRAMENAME
```
The frame directory must now be linked to the rust2 workspace directory.
```shell
cd /rust2/workspace/frames/
ln -s /${RUSTDIR}/workspace/frames/${FRAMENAME} .
```
To set up a frame in the database (`python`):
```shell
fid = '000A_UK_131313'
bl = ['000_12345_IW1', '000_12345_IW2', '000_12345_IW3']
dbq.make_new_frame_using_burstlist(bl, fid)
```
To check the set-up is succesful, try (`python`):
```shell
frame = dbq.query(Frame, id=fid)[0]
print(frame.id, frame.track)
```
If using the frame database for processing, it is good practice to update the database with the data download location (`python`).
```shell
dbq.update(Frame, {'downloaddir':'/data/T000'}, id=fid)
```

Now download some data (replace input flags). Once the data has been unzipped, remember to delete the zip files to save memory on the virtual machines.
```shell
master_download_script.py -f $FRAMENAME -d /data/T${TRACK}/
```
The MASTERDATE is now defined if using the database (`python`):
```shell
frame = dbq.query(Frame, id=fid)[0]
frame.master[0].date.strftime('%Y%m%d')
```
```shell
MASTERDATE=YYYYMMDD
echo $MASTERDATE
```

The final remaining step prior to processing is to create a DEM. Within the frame directory:
```shell
cd /workspace/frames/$FRAMENAME
make_frame_dem.py -d /data/SRTM -f $FRAMENAME -o DEM
```
Check the .bmp file in the output directory to confirm the DEM is correct (`eog DEM/*.bmp`).

### Processing overview
Now that SatSAR has been set up, you can start using it. The commands for each stage of the processing are summarised below:
```shell
make_images.py -d `pwd` -f $FRAMENAME -m $MASTERDATE
coreg.py -d `pwd` -f $FRAMENAME -m $MASTERDATE
make_interferograms.py -d `pwd` -f $FRAMENAME -s
```

A summary of the input argument placeholders:
| Placeholder | Description                                                    |
| ----------- | -----------                                                    |
| WORKDIR     | Working directory for the frame processing (`pwd`)             |
| FRAMENAME   | The name of the frame you want to process
| MASTERDATE  | The master date                                                |

# RapidSAR time-series processing
Following the initial single-master processing with SatSAR, we now run the rest of the time-series processing with RapidSAR. Within the processing directory, create a RapidSAR working directory. It is also worth running these commands in a `screen` as they can take a while.
```shell
mkdir RapidSAR
```
Prior to running the ingest step of RapidSAR, we need to generate some files in the `geo` directory:
```shell
get_lonlat.py -d `pwd` -f $FRAMENAME
get_look_angles.py -d `pwd` -f $FRAMENAME
```
If cropping the data down prior to running the time-series analysis, at this stage run (and use the output during the RapidSAR ingestion, also recommend saving to text file for future reference):
```shell
get_lonlatbounds_shp.py -s .shp
get_crop_area.py -d `pwd` -p top,bottom,left,right
```
To ingest the processed data into RapidSAR, run:
```shell
RapidSAR_ingest.py RSLC IFG data.h5 -f $FRAMENAME [-c top,bottom,left,right]
```
There will now be a single data.h5 file in the directory (to list its contents: `h5ls data.h5`).

## Coherence estimation
The next step is to use the sibling coherence method to accurately estimate the coherence on a pixel. Previous measures use a window surrounding the pixel to estimate the coherence. However this estimation breaks down if pixels are adjacent to non-similar pixels (e.g. the edge of buildings). Instead the sibling method identifies similar points within the image (up to 100 siblings per pixel) and uses these to estimate the coherence.
```shell
RapidSAR_find_siblings.py RSLC IFG data.h5 Coherence -f $FRAMENAME
```
This creates a .shp file (statistically homegenous pixels, i.e. siblings, not a QGIS format...) and a coh.h5 file. For more details on the sibling coherence method, see: _Spaans K, Hooper A. InSAR processing for volcano monitoring and other near‐real time applications (2016). JGR Solid Earth_.

## Unwrapping
For the unwrapping, we use the algorithm/program SNAPHU (Chen and Zebker, Stanford). This converts wrapped phase (-π to π) to continuous unwrapped phase (also in radians). The [-c] flag is the phase variance threshold (any pixels above this are considered noise).
```shell
RapidSAR_unwrap.py IFG Coherence data.h5 RSLC -f $FRAMENAME
```
This process creates a file uw.h5 within the RapidSAR directory. The uw.h5 file contains two unwrapped datasets. The first is the "high resolution" (full) unwrapped interferograms. The second is the lower resolution "rural" dataset. The rural dataset is multi-looked (downsampled) with 3 looks in range and 12 looks in azimuth (Sentinel-1 data is acquired in non-square pixels, this ratio converts to approximately square). 

The rural data has a more coarse resolution which makes it easier to handle (look at the differences in file size of the output velocity maps in the next step). The pixel size is approximately 50 m × 50 m. There is currently an issue with rural datasets in that they are affected by fading phase-bias (see: _Ansari, H., De Zan, F. and Parizzi, A. Study of Systematic Bias in Measuring Surface Deformation with SAR Interferometry (2020) IEEE_). The source of the bias remains unknown, however it results in the apparent subsidence of rural pixels and is more pronounced in short temporal baseline interferograms.

## Time-series inversion
The time-series inversion is run on both the rural and high-resolution datasets. An overly simplified description of the time-series inversion is that it is solving a least-squares inversion for the displacement of each image relative to the first date in the time-series, from the network phase displacements in each interferogram in the interferometric network. This generates a displacement time-series which can then be used to derive the average velocity of each pixel in the line-of-sight (LOS). In practice this is more complex, including referencing the time-series in space and time, running the inversion multiple times to detect unwrapping errors, and mapping out islands in the track extent using the DEM. We run this in two stages (rural resolution, then full resolution):
```shell
multicore_ts.py Coherence data.h5 ${FRAMENAME}-rural-vel.h5 -f $FRAMENAME -l
multicore_ts.py Coherence data.h5 ${FRAMENAME}-vel.h5 -f $FRAMENAME -z ${FRAMENAME}-rural-vel.h5
```

## Post-processing
```shell
multicore_postproc.py -d ${FRAMENAME}-rural-vel.h5 -i ${FRAMENAME}-vel.h5 -p data.h5 -f $FRAMENAME
```
SatSense 'standard' post-processing applies several steps to the "Cumulative_Displacement" dataset to create new datasets in the `*-vel.h5` files.
* Remove long-wavelength signals with a 30 km Gaussian filter ("Cumulative_Displacement_Filt").
* APS filter: using the assumption that atmosphere should be random in time (high-pass) but correlated in space (low-pass)("Cumulative_Displacement_APS").
* Smooth displacement time-series in time using a triangular weighted smoothing ("Cumulative_Displacement_TSmooth").

This also calculates the RMS error for the time-series. It is possible to skip options with flags (use input "SKIP"), however then watch out for default inputs in subsequent steps.

An additional required post-processing step is the reliable pixels estimation. This recognises pixels affected by rural bias and removes them. To run this:
```shell
get_reliable_pixels.py -d ${FRAMENAME}-rural-vel.h5 -i ${FRAMENAME}-vel.h5 -p data.h5 -f 0
```

## Data export to QGIS
After this, it is possible to generate shapefiles for export to QGIS:
```shell
rsar_extract.py -d ${FRAMENAME}-vel.h5 -e data.h5 -o $FRAMENAME -r REF.shp -p linear,seasonal
rsar_extract.py -d ${FRAMENAME}-rural-vel.h5 -e data.h5 -o ${FRAMENAME}_RURAL -r REF.shp -p linear,seasonal
```
