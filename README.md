# PALM_static
**under development**

The geotiff files are important input for static driver. However, it is impossible to process all the geoinformation in a standard way. Here we present scripts to genearte static drivers for PALM simulation. We provide an interface for users to download geospatial data globally, while users can also provide their own geospatial data in `tif` format. The script will prepare all input files for the configured simulation domains and then generate static drivers. Hopefully these tools can make PALM users' lives easier.

## How to run?
The main script is `run_config_static.py`. To run this script, a namelist file is required. The namelist for each case should be  `JOBS/case_name/INPUT/namelist.static-case_name`.

### namelist 
The namelist requires PALM domain configuration and geotiff filenames from users. The domain configuration is similar to variables used in PALM. 

Users must specify:
```
[case]
case_name         -  name of the case 
origin_time       -  date and time at model start*
default_proj      -  default is EPSG:4326. This projection uses lat/lon to locate domain. This may not be changed.
config_proj          -  projection of input tif files. We recommend users use local projection with units in metre, e.g. for New Zealand users, EPSG:2193 is a recommended choice.

[domain]
ndomain           -  maximum number of domains, when >=2, domain nesting is enabled  
centlat, centlon  -  centre latitude and longitude of the first domain. Note this is not required for nested domains  
nx                -  number of grid points along x-axis  
ny                -  number of grid points along y-axis  
nz                -  number of grid points along z-axis  
dx                -  grid spacing in meters along x-axis  
dy                -  grid spacing in meters along y-axis  
dz                -  grid spacing in meters along z-axis  
z_origin          -  elevated terrain mean grid position in meters (leave as 0.0 if unknown)  
ll_x              -  lower left corner distance to the first domain in meters along x-axis   
ll_y              -  lower left corner distance to the first domain in meters along y-axis   

[geotif]          -  required input from user; can be provided by users in the INPUT folder or "online"
sst               -  input for water temperature
dem               -  digital elevation model input for topography
lu                -  land use classification  
resample_method   -  method to resample geotiff files for interpolation/extrapolation

# if NASA API is used format in YYYY-MM-DD
# SST date should be the same as the orignin_time

## No need to change start/end dates for NASA SRTMGL1_NC.003 
dem_start_date = '2000-02-12',  
dem_end_date = '2000-02-20',
## start/end dates for land use data set
lu_start_date = '2020-10-01',
lu_end_date = '2020-10-30',

[urban]             - input for urban canopy model; can leave as "" if this feature is not included in the simulations, or provided by user; or online from OSM
bldh                - input for building height 
bldid               - input for building ID
pavement            - input for pavement type
street              - input for building ID

[plant]           - input for plant canopy model; can leave as "" if this feature is not included in the simulations, or provided by user
sfch              - input for plant height; this is for leave area density (LAD)
```

**below needs to be edited (20/09/2022)**

The **required** fields for tif files are `dem` and `lu`. A lookup table (in `raw_static` folder) is required to convert land use information to PALM recognisable types. Here we used New Zealand Land Cover Data Base (LCDB) v5.0. Our lookup table `nzlcdb_2_PALM_num.csv` is available in `raw_static` folder. 

_The `origin_time` setting is similar to `origin_date_time` in [PALM documentation](https://palm.muk.uni-hannover.de/trac/wiki/doc/app/initialization_parameters#origin_date_time). This variable is required in static drivers, but will not be used in PALM simulation. Rather the date time should be specified in PALM's p3d namelist. The sunset/sunrise time is affected by lat/lon attributes in the static driver._

**Note: when no urban input is used, the vegetation type is set to 18 and the albedo type is set to 33 for urban area specified in land use files.**

For other tif file fileds, if users do not have files available, they should leave the file names empty as `"",`. The script will automatically read the "empty" tif file (`empty.tif`) provided in `raw_static`. 

Note that if the provided `empty.tif` causes any error (usually due to insufficient grid cells). Users may create their own empty tif file based on their own tif files using `create_empty.py`:
```
python create_empty.py [input tif file]
```

Two namelist examples are given in `create_static` folder - one for the most simple configuration (`namelist.static.simple`) and the other for all the available features at present (`namelist.static.all_feature`).

#### input tif files explained
We processed our own geotiff files using the GIS tools before using the python scripts here.  
- `bldh` refers to building height. This is calculated using the difference between digital surface model (DSM) and DEM. The building height is extracted using OpenStreet Map (OSM) building outlines.  
- `bldid` refers to buliding ID (available in OSM).   
- `street`refers to street type (available in OSM).  
- `sfch` refers to surface object height excluding buildings. This is calculated using the difference between digital surface model (DSM) and DEM. Buildings are excluded using building outlines available in OSM.  

**Note: building type information is not available in New Zealand, and hence one building type is assigned for all buildings.**   
  
Variables in the static driver here are not inclusive. Users may refer to PALM input data standard or Heldens et al. (2020).

_Heldens, W., Burmeister, C., Kanani-Sühring, F., Maronga, B., Pavlik, D., Sühring, M., Zeidler, J., and Esch, T.: Geospatial input data for the PALM model system 6.0: model requirements, data sources and processing, Geosci. Model Dev., 13, 5833–5873, https://doi.org/10.5194/gmd-13-5833-2020, 2020._


### geotiff files requirements (To Do: new readme for this section needed)
- Users may put their geotiff files in `create_static/raw_static`. 
- The geotiff files must have the same projection. 
- The geotiff files must have the same resolution as desired in PALM simulation, e.g. for a 10 m simulation, the geotiff files resolution must be 10 m. 

Users have their own geotiff files ready but the resolution and/or projection do not satisfy the requirements. We provide a python script `prep_tif.py` to reproject and resample geotiff files in `prep_static` folder.   
Users may provide their own tif files in `prep_static/tiff/` and run `prep_tif.py` for repreojection and resample:  
```
python prep_tif.py [infile] [out EPSG projection] [outfile prefix] [resolution list]
```

Once all geotiff files are ready, they can be linked from `prep_static/tiff` into `create_static/raw_static`:
```
ln -sf prep_static/tiff/*.tif create_static/raw_static/.
```

### run the main script
Now if users have all geotiff files ready, they may run the main script:
```
python run_config_static.py [namelist_file]
```

The script should print some processing information and create the desired static files, which can be found in `static_files`. Each domain will also have 
1. its own geotiff file created in `static_files` for georeferences.
2. its own cfg file created in `cfg_files` for future reference in e.g. WRF4PALM.

### visualise domain on OSM
Users may visualise domain by running `visualise_PALM_domains.py`:
```
python visulalise_PALM_domains.py [namelist_file]
```
This can be done before static files are created.

### flat terrain and precursor run
Once a static driver is used, all the PALM domains in the simulation requires static drivers. In case a flat terrain static driver and/or precursor run static driver are required, users may run `static_to_flat.py`. 
```
python static_to_flat.py [static_file] [nx,ny]
```

### water temperature
The water temperature is derived using monthly mean of ERA5 SST data for 2019. The month of the year is derived from `origin_time` in the namelist. The location to take SST data depends on `centlat` and `centlon` in the namelist.

Note that this requires no urban variables (e.g. buildings and streets) in the input static driver. If precursor run is not required, users do not need to specify `nx` and `ny`.

--------------------------------------------------------------------------------------------  
We have been trying to add more comments and more instructions of the scripts. However, if there is anything unclear, please do not hesitate to contact us. 

Dongqi Lin (dongqi.lin@pg.canterbury.ac.nz)  
Jiawei Zhang (Jiawei.Zhang@scionresearch.com)  

@ Centre for Atmospheric Research, University of Canterbury

--------------------------------------------------------------------------------------------
## End of README












