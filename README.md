
# Visual Code: Weather Open Data Tutorial :globe_with_meridians: :earth_americas:


Among the various weather indicators that NOAA and NASA collect and model, Precipitable Water (PWAT) is a crucial variable. PWAT measures the total amount of water vapor in the entire atmosphere above a specific location on Earth's surface. Essentially, it represents the potential amount of water that could be extracted from the atmosphere as rain or snow. When PWAT values are high, the atmosphere is rich with moisture, making it possible for storms to produce large amounts of precipitation. Conversely, low PWAT values indicate a drier atmosphere, reducing the likelihood of significant rainfall.

It’s essential to remember that PWAT by itself doesn’t determine whether precipitation will happen—it only shows the potential for it, depending on the occurrence of other atmospheric processes like thunderstorms. However, it's still a critical measurement collected by satellite instruments and is heavily used in NOAA’s weather forecasting.

In this tutorial, we'll be using two main tools: the Terminal, which allows direct interaction with NOAA’s web interface and database, to work with weather data encoding and formats, and a web code editor (Visual Studio Code in this case, though any similar editor will work) to project this data interactively on the web.

## Tutorial Breakdown:
The tutorial is divided into two parts: 
- **Part 1** We’ll go through processing and visualizing PWAT data from the 0.25-degree Global Forecast System (GFS). Given how atmospheric rivers and water vapor move dynamically, shaping weather patterns, we’ll explore how to create an animated video map of this data spanning 12 days, revealing complex temporal patterns.
- **Part 2** We’ll build a simple web interface that projects the animated video globally on a spherical surface, enabling interactive exploration of the data.

## Part 1: Processing and Visualization of PWAT Data

### Step 1: Preliminaries

Before diving into the data, you'll need to set up your environment by installing the necessary tools. Here's a quick overview of the main libraries and utilities we'll be using, along with instructions on how to download and install each one.

- **GDAL**: GDAL (Geospatial Data Abstraction Library) is used for translating and manipulating raster and vector geospatial data formats. 
  - **Installation**: You can install GDAL using Homebrew on macOS or through a package manager on Linux.
    ```bash
    brew install gdal
    ```
  - Alternatively, you can install it via `pip` for Python:
    ```bash
    pip install gdal
    ```

- **Parallel**: GNU Parallel is a shell tool for executing multiple jobs in parallel, making it efficient for processing large datasets.
  - **Installation**: On macOS, you can install it using Homebrew:
    ```bash
    brew install parallel
    ```
  - On Linux, you can install it through your package manager, such as:
    ```bash
    sudo apt-get install parallel
    ```

- **rasterio**: Rasterio is a Python library designed for reading and writing geospatial raster data. It provides a clean API that integrates well with NumPy.
  - **Installation**: Install it via `pip`:
    ```bash
    pip install rasterio
    ```

- **gribdoctor**: Gribdoctor is a set of utilities for handling quirks in GRIB (General Regularly-distributed Information in Binary) weather data files.
  - **Installation**: You can install `gribdoctor` using `pip`:
    ```bash
    pip install gribdoctor
    ```

- **ffmpeg**: FFmpeg is a powerful multimedia framework used for recording, converting, and streaming audio and video files. It is essential for creating animations from image sequences.
  - **Installation**: On macOS, install FFmpeg using Homebrew:
    ```bash
    brew install ffmpeg
    ```
  - On Linux, install it via your package manager:
    ```bash
    sudo apt-get install ffmpeg
    ```

- **wget**: Wget is a command-line utility for downloading files from the web, which we'll use to retrieve data from NOAA’s servers.
  - **Installation**: On macOS, install it using Homebrew:
    ```bash
    brew install wget
    ```
  - On Linux, install it through your package manager:
    ```bash
    sudo apt-get install wget
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



### Step 1: Projecting the Video onto a Sphere


### Step 2: Creating a Web Interface

:earth_americas:

### Step 3: Extending the Tutorial to Other Variables

This tutorial is focused on PWAT, but the process can easily be adapted to visualize other meteorological variables from NOAA’s archives. Variables such as temperature, wind speed, and pressure are available in similar formats and can be processed in the same way to create dynamic weather visualizations.

---

