
# Visual Code: Weather Open Data Tutorial

This tutorial will guide you through extracting, processing, and visualizing Precipitable Water (PWAT) data from NOAA's Global Forecast System (GFS) model. PWAT is a key metric in weather forecasting, indicating the potential amount of water vapor in the atmosphere that can condense into precipitation, such as rain or snow. While PWAT doesn’t guarantee precipitation, it is essential for understanding the moisture content in the atmosphere and its potential to fuel storms.

The tutorial is divided into two parts: 
- **Part 1** covers processing and visualizing PWAT data, including creating animated maps that reveal dynamic temporal patterns over 12 days.
- **Part 2** walks you through projecting the animated video onto a global 3D sphere and creating an interactive web interface.

## Part 1: Processing and Visualization of PWAT Data

### Step 1: Preliminaries

Before diving into the data, you’ll need to set up your environment and install some essential tools. We will use the following libraries and utilities:

- **GDAL**: A library for translating raster and vector geospatial data formats.
- **Parallel**: A shell tool to execute jobs in parallel for efficient processing.
- **rasterio**: A Python library for reading and writing geospatial raster data.
- **gribdoctor**: Utilities to handle quirks in GRIB (General Regularly-distributed Information in Binary) weather data files.
- **ffmpeg**: A powerful multimedia framework to record, convert, and stream audio and video.
- **wget**: A command-line utility to download files from the web.

You can install the required utilities using Homebrew on macOS:

```bash
brew install parallel wget
```

### Step 2: Project Directory Setup

Start by creating a dedicated directory for your project to organize your files:

```bash
mkdir my_weather_project
cd my_weather_project
```

Now, create subdirectories for storing raw data, processed data, and output images:

```bash
mkdir data_raw global global_tif global_color global_png
```

Your project directory structure should look like this:

```
my_weather_project/
├── data_raw/
├── global/
├── global_tif/
├── global_color/
└── global_png/
```

### Step 3: Download PWAT Data from NOAA

NOAA provides PWAT data via their web interface. Here’s an example of the URL structure to download PWAT data for a specific date and time:

```text
https://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t06z.pgrb2.0p25.anl&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.20240815%2F06%2Fatmos
```

This URL contains parameters such as:
- **time**: The model run hour (e.g., 06z).
- **date**: The date (e.g., 20240815).

You can automate the downloading of PWAT data for multiple dates and times using the GNU Parallel tool:

```bash
parallel --header : wget '"https://nomads.ncep.noaa.gov/cgi-bin/filter_gfs_0p25.pl?file=gfs.t{time}z.pgrb2.0p25.anl&var_PWAT=on&leftlon=0&rightlon=360&toplat=90&bottomlat=-90&dir=%2Fgfs.202408{date}%2F{time}%2Fatmos"' -O data_raw/202408{date}{time}.grib2 ::: date {01..31} ::: time {00,06,12,18}
```

Explanation:
- **Filename Pattern**: The `-O` option specifies the output filename, such as `data_raw/202408{date}{time}.grib2`.
- **parallel**: Runs multiple `wget` commands in parallel, speeding up the download process.

This command will download all PWAT data for the specified dates and times into the `data_raw` directory.

### Step 4: Convert GRIB2 to Global Format

Once the data is downloaded, convert each `.grib2` file in the `/data_raw` directory to a global format using the following command:

```bash
for file in data_raw/*.grib2; do cdo -f grb2 -sellonlatbox,-180,180,-90,90 $file global/$(basename $file); done
```

### Step 5: Convert GRIB2 to TIFF Format

Next, convert each global `.grib2` file to a TIFF format:

```bash
for file in global/*.grib2; do gdal_translate -of GTiff "$file" "global_tif/$(basename "$file" .grib2).tif"; done
```

### Step 6: Colorize PWAT Data

To make the PWAT data visually informative, apply a color ramp. Save the following color values in a text file named `color_ramp.txt`:

```text
10.0 255 255 255
18.0 204 255 204
26.0 153 255 153
34.0 102 255 102
42.0 51  255 51
50.0 0   255 0
58.0 0   204 51
66.0 0   153 102
74.0 0   102 153
82.0 0   51  204
90.0 0   0   255
```

This color ramp maps PWAT values to specific colors, making it easier to visualize variations in moisture content. Now, apply the color ramp to all the TIFF files:

```bash
parallel gdaldem color-relief {} color_ramp.txt global_color/{/.}.tif ::: global_tif/*.tif
```

### Step 7: Convert to PNG Format

To prepare the data for animation, convert the colorized TIFF files to PNG format:

```bash
parallel rio convert {} global_png/{/.}.png --format PNG ::: global_color/*.tif
```

### Step 8: Create an Animation

Finally, animate the sequence of PNG images into a video using `ffmpeg`:

```bash
ffmpeg -r 30 -pattern_type glob -i 'global_png/*.png' -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" -c:v libx264 -preset veryslow -crf 18 -pix_fmt yuv420p animation.mp4
```

This command creates a smooth 30 frames per second (fps) video, ready for further exploration.

## Part 2: Creating an Interactive 3D Globe

Now that you’ve created an animated video, you can project it onto a 3D sphere and create an interactive web-based interface using tools like **ESRI** and **ArcGIS**.

### Step 1: Projecting the Video onto a Sphere

Using the ESRI and ArcGIS interface, you can take the animated video and project it onto a 3D globe. By mapping the video frames to the surface of a sphere, you can simulate the movement of water vapor around the Earth’s atmosphere.

The interactive 3D globe allows users to explore temporal and spatial changes in PWAT data dynamically. You can rotate the globe, zoom in on specific regions, and analyze the weather patterns over time.

### Step 2: Creating a Web Interface

To make your 3D globe accessible to others, you can embed it in a simple web interface. With web technologies like HTML, CSS, and JavaScript, you can create an interactive visualization that users can manipulate directly in their browsers.

### Step 3: Extending the Tutorial to Other Variables

This tutorial is focused on PWAT, but the process can easily be adapted to visualize other meteorological variables from NOAA’s archives. Variables such as temperature, wind speed, and pressure are available in similar formats and can be processed in the same way to create dynamic weather visualizations.

---

This tutorial provides a step-by-step guide to extracting, processing, and visualizing PWAT data, enabling you to explore complex weather patterns in a visually engaging way. By the end of this tutorial, you’ll have an animated map of global PWAT data and an interactive 3D globe, giving you the tools to understand and analyze atmospheric moisture content dynamically.
