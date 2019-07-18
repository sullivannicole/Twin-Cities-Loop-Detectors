:: --------------------------------------------------------------

# BACKGROUND

:: --------------------------------------------------------------

Code in this repo processes data collected from Minnesota Department of Transportation (MnDOT) loop detectors installed on the Minnesota Freeway system in 30-second interval measurements of occupancy and volume, data which are pushed daily to a public [JSON feed](http://data.dot.state.mn.us:8080/trafdat/).  To use these data, the following process was used:

1. Data are extracted from the JSON feed and transformed into dataframes
2. Performance measures are calculated from the dataframes
3. The spatial configuration of the detectors is downloaded as an XML, transformed into a dataframe and then further transformed into a SpatialLinesDataFrame, which is written out as a shapefile using ESRI Shapefile as the driver.
The project itself was carried out by the metropolitan planning organization (MPO) for the region, the [Metropolitan Council](metrocouncil.org).  Under federal law, MPOs are responsible for long-range planning for transportation in the region, which includes the [Congestion Management Process](https://metrocouncil.org/Transportation/Planning-2/Key-Transportation-Planning-Documents/Congestion-Management-Process.aspx) (CMP) and [Travel Demand Management](https://metrocouncil.org/Communities/Planning/TOD/Planning-Fundamentals/Parking-Travel-Demand-Management.aspx) (TDM), and the Council uses a number of performance measures (including those calculated using this code) to help target these mitigative strategies to pain points on the regional system.  For questions regarding this project, contact Nicole Sullivan (nicole.sullivan@metc.state.mn.us).

:: --------------------------------------------------------------

# EXTRACTION

:: --------------------------------------------------------------

The code contained in *Loop Detector Data from JSON Feed.Rmd* extracts data from [MnDOT's JSON feed](http://data.dot.state.mn.us:8080/trafdat/) (sensor level URL format: http://data.dot.state.mn.us:8080/trafdat/metro/yyyy/yyyymmdd/ssss.x30.json), which updates daily with 30-second interval measurements of occupancy and volume (explained further in the speed processing section) at the sensor level.

## Pull options
There are 4 pull options:  sensor_pull, corridor_pull_slice, corridor_pull or corridor_pull_par.  Range of time for data pull needs to be specified in each function (via start_date and end_date arguments); **there is no default date range**.

## Pull recommendations
* Sensor_pull:  pulls data for a single sensor for date range specified
* Corridor_pull_slice:  pulls data for a corridor for a few sensors at a time.  This function is great for testing speed of processing.  Recommended use with microbenchmark.
* Corridor_pull:  pulls data for a corridor, without parallel-run specifications. This is **not recommended for Windows OS,** which runs sequentially by default.  Use corridor_pull_par, which will speed up extraction by approximately the number of cores on the machine (i.e. a 4-core Windows machine will run extraction at ~4X the processing speed when using corridor_pull_par than when using corridor_pull).
* Corridor_pull_par:  pulls data for a corridor with parallel-run specifications - This is **recommended for Windows OS**.

## Output
Output is one file for each sensor for entire date range specified.

## Output Size
A complete year's worth of data for volume or occupancy for one sensor usually results in a file that is around ~30-31KB.

## Run-time
Approximate time to pull one sensor's and one extension's (v or c for volume or occupancy, respectively) data across a year on a Mac is 1.33 minutes.

## Compatibility
This code has been tested and validated on Mac iOS and Windows OS.

## Missing Data
Code has built-in error handling in the case that an entire day's worth of data (for either volume or occupancy) is missing for a sensor (a row is populated for the date in which time is listed as "Entire day missing").  If only part of a day is missing at a location, these are recorded as NULLs in data, and thus automatically differentiated from actual measured zeros (appear as '0' in the data).

## Required libraries
library(tidyverse)
library(jsonlite)
library(stringr)
library(timeDate)
library(lubridate)
library(rlang)
library(parallel)
library(data.table)
library(microbenchmark)
library(foreach)
library(doParallel)

:: --------------------------------------------------------------

# SPEED AND PERFORMANCE MEASURES CALCULATIONS

:: --------------------------------------------------------------

## Volume and Occupancy Processing

Once, data is extracted from the JSON feed, code in the *Calculating Measures.Rmd* takes the full 1 million+ length volume dataset and 1-million+ length occupancy dataset pulled from JSON feed for the same sensor, and converts them to one dataset containing estimated speed of traffic at the sensor over 10-minute intervals.  The speed equation is:

$$\frac{Vehicles Per Hour*Vehicle Length}{5280*Occupancy Percentage} $$

or
$$\frac{Flow}{Density} $$

The 'Vehicles Per Hour' variable is calculated by summing all the vehicles over the 15-minute interval, and then multiplying that by four.  The 'Vehicle Length' variable is a static field in the sensor configuration dataset.  The 'Occupancy Percentage' variable is calculated by summing all the occupancy values over the 15-minute interval, and then dividing by 54,000 (1,800 scans in 30 seconds*30 periods in 15-minute interval).

**A note on the term 'occupancy'**

The term 'occupancy' does not here refer to the occupants of a vehicle but rather the occupancy of the sensor, or how long the sensor was 'occupied'.  In a 30-second time period, 1800 scans are produced (60 per second), and each scan is binary:  either the sensor is occupied or not.  Therefore, a sensor occupied for 1 second within the 30-second time period would have a value of 60.  Raw occupancy values can be converted to percentages:

$$\frac{Occupancy}{1800}*100% $$
The resulting percentage is the percentage of time in that 30 seconds that the sensor was 'occupied'.

**A note on interpolation**

Where nulls exist (vs. a zero measurement), it is assumed the connection was disrupted and no measurement was taken.  For 15-minute intervals where other values exist, the nulls are interpolated with the **average** of the other values within the interval.  Impossible values are also interpolated with the mean of the interval.  Impossible values are raw occupancy values greater than 1800 (given that only 1800 scans are taken in a 30-second period).  If an entire interval contains only nulls, it is converted to 'NA' and no values within the interval are interpolated.

Note that a variable is created containing the percentage of nulls/impossible values in that interval; therefore, one can choose to exclude intervals with interpolation rates above a certain threshold of choice (eg if more than, say, 30% of the data is missing).

## Required libraries

library(tidyverse)
library(jsonlite)
library(stringr)
library(timeDate)
library(lubridate)
library(rlang)
library(parallel)
library(data.table)
library(microbenchmark)
library(foreach)
library(doParallel)

:: --------------------------------------------------------------

# SPATIAL DATA AND CONFIGURATION OF DETECTORS

:: --------------------------------------------------------------

The code in the *XML Conversion and Distance Calculations.Rmd* is used to convert the spatial configuration of detectors first to a dataframe.  The dataframe is used both to calculate distances between the upstream detector and the detector of interest (for approximation of vehicle miles traveled, VMT, on the network for a given period).  The dataframe is also converted to a SpatialLinesDataFrame which is then written out to a shapefile.  While the dataframe contains lat/lons that can be used for visualization purposes, a shapefile is more effective for visualization of lines in Tableau (as it eliminates the auto-borders created).

## Required libraries

library(stringr)
library(tidyverse)
library(xml2)
library(data.table)
library(geosphere)
