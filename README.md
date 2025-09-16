[![GitHub Pages](https://img.shields.io/badge/GitHub-Pages-blue?logo=github)](https://ddols.github.io/Step_by_step_mapping_tutorial_in_R/)

Tutorial: Mapping localities in R with Lambert projection
================
D. Dols-Serrate
2025-08-06

- [Basic requirements](#basic-requirements)
- [Introduction](#introduction)
- [Step 1: Loading Required R
  Packages](#step-1-loading-required-r-packages)
- [Step 2: Preparing the Locality and Map
  Data](#step-2-preparing-the-locality-and-map-data)
- [Step 3: Defining the Map Projection and
  Viewport](#step-3-defining-the-map-projection-and-viewport)
- [Step 4: Building the Map with
  `ggplot2`](#step-4-building-the-map-with-ggplot2)
- [Final Remarks](#final-remarks)


# Basic requirements

- `R 4.1.0` or beyond. This tutorial was run on `R 4.5.1`.

- Preferably, at least one finger (optional).

# Introduction

This document provides a step-by-step tutorial for creating a
high-quality map of sampled localities in Europe (although it could be
anywhere) using R and the `ggplot2` package. We will start with raw
latitude and longitude coordinates, project them onto a proper
cartographic projection (Lambert Azimuthal Equal-Area), and customize
the final map with colors, shapes, labels, and other cartographic
elements. If everything runs smoothly, the expected result should look
like the example below.

<div class="figure" style="text-align: center">

<img src="Fig1_WM_Lambert_minimal.png" alt="**Fig. 1.** Example of a map obtained with the methodology explained in this tutorial" width="80%" />
<p class="caption">

**Fig. 1.** Example of a map obtained with the methodology explained in
this tutorial
</p>

</div>

The workflow is based on modern R spatial packages (`sf`,
`rnaturalearth`, `ggrepel`) which provide a powerful and coherent system
for spatial analysis and visualization. Feel free to make any changes
and/or improvements on the code.

# Step 1: Loading Required R Packages

The first step in any R script is to load the necessary “libraries” or
“packages” that contain the functions we need. We use the `library()`
command for this. If you are running this for the first time, you may
need to install these packages.

**To Install (run this in your R console *once*):**

`install.packages(c("sf", "ggplot2", "rnaturalearth", "ggrepel", "ggspatial"))`

``` r
# Load the libraries for this session
library(sf)            # The modern framework for handling spatial data
library(ggplot2)       # The core package for creating all plots
library(rnaturalearth) # A great source for high-quality, free background map data
library(ggrepel)       # For adding non-overlapping text labels
library(ggspatial)     # For adding map elements like scale bars and north arrows
```

If problems arise when installing `rnaturalearth`, then try the
following steps:

1.  `install.packages("devtools", repos = "https://cloud.r-project.org/")` -
    Facilitates installation of R packages from github and other
    repositories

2.  `library(devtools)`

3.  `devtools::install_github("ropensci/rnaturalearthhires")` - Installs
    the troubling package from github repository

# Step 2: Preparing the Locality and Map Data

Next, we need to prepare our two main sources of data:

1.  **The locality data:** Our list of sampling points, including their
    coordinates and metadata.

2.  **The background map:** A map of the world that will serve as the
    canvas for our points.

### 2.1: Defining the Locality Data

In this tutorial we will build the `data.frame` from scratch, but
alternatively we could always resort to build it somewhere else (e.g.,
`Excel`, `LibreOffice`, etc.) or obtain it from a database, and then
load it using the following piece of code:

- `localities_df <- data.table::fread("my_localities.csv")`

or alternatively:

- `localities_df <- read.csv("my_localities.csv")`

When loading the data from a Comma-Separated Values (CSV) file, it is
best to ensure it is saved in the same directory as this R Markdown
document. We use the `fread()` function from the `data.table` package
because it is fast and robust.

We start by creating a standard R `data.frame` with our longitude,
latitude, and metadata. In this case, we will label each locality
numerically from 1 to *n* (`label =` variable) and will add information
concerning the species found in each location (`Data =` variable). If
you are building your `data.frame` directly in R make sure that the
order in which the localities and the metadata are entered in their
pertinent vectors (the `c(...)` thingy) is the same. We then will
convert this `data.frame` into a special `sf` (simple features) spatial
object. This tells R which columns represent coordinates and what their
initial Coordinate Reference System (CRS) is. We use `crs = 4326`, which
is the standard code for WGS84 latitude/longitude.

If you opted for building the `data.frame` in R, the following can be
used as a rough example of how you could do it.

``` r
example_df <- data.frame(
  lon = c(9.11, 15.13, 11.46, 18.12, 10.04),      # Longitudinal coordinates
  lat = c(33.08, 39.96, 32.03, 35.86, 37.76),     # Latitudinal coordinates
  label = as.character(1:5),                      # Locality numbers or names. The later as in 'label = c("A", "B", "C" ...)
  Data = c("Sp.A","Sp.A","Sp.C","Sp.B","Sp.A")    # Add as many metadata columns as your data require. Here, data regarding the species affiliation is being added
)

# Convert the data frame into an 'sf' spatial object
example_sf_with_data <- st_as_sf(example_df, coords = c("lon", "lat"), crs = 4326)
```

### 2.2: Fetching the Background Map

For the sake of this tutorial, we will work with occurrence data of
several allochthonous species. Hence, we will start by creating random
points within the borders of the European continent. We will use the
`rnaturalearth` package to get high-quality country outlines. We specify
`returnclass = "sf"` to ensure the map data is in the same format as our
locality points.

``` r
# --- A. Set the number of random locations you want to generate ---
num_locations <- 35 # You can change this number to whatever you need

# --- B. Get high-resolution country outlines for the European continent ---
# We use rnaturalearth to get country data, then filter for Europe

europe_map <- ne_countries(scale = "large", continent = "Europe", returnclass = "sf")

# To sample from the entire landmass, we merge all individual country polygons
# into a single, unified shape.
europe_landmass <- st_union(europe_map)

# --- C. Generate random points constrained *within* the European landmass shape ---
# The st_sample function is perfect for this task.

random_points_sf <- st_sample(europe_landmass, size = num_locations, type = "random")

# --- D. Extract the longitude and latitude coordinates from the random points ---
# st_coordinates gives us a matrix of X (longitude) and Y (latitude) values
random_coords <- st_coordinates(random_points_sf)

# --- E. Create the final data frame in the same format as the example_df ---
# This is now your primary data frame for plotting.
random_european_locations_df <- data.frame(
  lon = random_coords[, "X"],
  lat = random_coords[, "Y"],
  # We'll add some placeholder metadata so the rest of the script works
  label = as.character(1:num_locations),
  Data = sample(c("Achamo torchicum", "Kimori treeckoi", "Mizugorou mudkipus"), size = num_locations, replace = TRUE)
)

# Convert the data frame into a proper 'sf' object.
random_points_sf_with_data <- st_as_sf(random_european_locations_df, 
                                       coords = c("lon", "lat"), 
                                       crs = 4326)

# You can inspect the first few rows to see the result
head(random_points_sf_with_data)
```

    ## Simple feature collection with 6 features and 2 fields
    ## Geometry type: POINT
    ## Dimension:     XY
    ## Bounding box:  xmin: 97.8206 ymin: 50.02749 xmax: 165.8918 ymax: 72.51251
    ## Geodetic CRS:  WGS 84
    ##   label               Data                  geometry
    ## 1     1    Kimori treeckoi POINT (114.3639 72.51251)
    ## 2     2    Kimori treeckoi  POINT (97.8206 50.02749)
    ## 3     3 Mizugorou mudkipus POINT (116.8413 68.60778)
    ## 4     4   Achamo torchicum   POINT (123.306 66.4949)
    ## 5     5    Kimori treeckoi POINT (165.8918 63.76939)
    ## 6     6    Kimori treeckoi POINT (119.2057 54.88758)

Needless to say, `rnaturalearth` has other options and if wanted to keep
it simple we could always get the outlines of the ‘whole world’ and then
crop to the desired resolution (keep reading).

``` r
# Get high-resolution country outlines
world_map <- ne_countries(scale = "large", returnclass = "sf")
```

# Step 3: Defining the Map Projection and Viewport

A crucial step in creating a true map is projecting our spherical globe
data onto a flat 2D surface. We will use the **Lambert Azimuthal
Equal-Area (LAEA)** projection, which is well-suited for Europe
(`EPSG:3035`). We also define the rectangular area (the “bounding box”)
we want our final map to show, using standard latitude/longitude
coordinates. Whenever you want to adjust the limits of your map just
change the values for `xmin`, `ymin`, `xmax`, and `ymax`. In this case,
we chose values to crop Europe and a little bit of its surroundings.

``` r
# Define the Lambert projection we want to use for the final map
crs_lambert <- "EPSG:3035"

# Define the bounding box for our final map in Lat/Lon coordinates
map_bounds_latlon <- st_bbox(c(xmin = -15.3, ymin = 30.3, xmax = 50.3, ymax = 80.5), crs = 4326)
```

# Step 4: Building the Map with `ggplot2`

Now we will build our map layer by layer using `ggplot2`. This “grammar
of graphics” approach makes it easy to build up complex plots.

### Explanation of the Layers:

1.  **`geom_sf(data = world_map, ...)`**: The base layer, drawing the
    countries.
2.  **`geom_sf(data = random_points_sf, ...)`**: Draws the locality
    points. Here, we map the **color** and **shape** aesthetics to our
    `Data` column.
3.  **`geom_text_repel(...)`**: Adds the numerical labels near each
    point, cleverly repositioning them to avoid overlap.
4.  **`scale_color_manual()` & `scale_shape_manual()`**: These give us
    full control over which colors and shapes are used for each category
    in our data.
5.  **`coord_sf(...)`**: This is the engine of our map. It reprojects
    all data to our target Lambert CRS and then zooms the view to our
    predefined bounding box.
6.  **`annotation_scale()`**: Adds a cartographically correct scale bar.
7.  **`theme()`**: Fine-tunes the visual appearance of non-data elements
    like the legend.

``` r
# The main ggplot call to build the map
ggplot() +
  # Layer 1: The background map
  geom_sf(data = world_map, fill = "gray80",     # We define the color of inland areas
          color = "gray60") +                    # We define the color of the borders  
  
  # Layer 2: European points with aesthetics mapped to the 'Data' column
  geom_sf(
    data = random_points_sf_with_data,
    aes(color = Data, shape = Data),
    size = 4,        
    stroke = 1.5     
  ) +
  
  # Layer 3: Add labels to the European points
  geom_text_repel(
    data = random_points_sf_with_data,
    aes(geometry = geometry, label = label), # Note we use the 'label' column here
    stat = "sf_coordinates", 
    size = 3,
    nudge_y = -12000    # Nudge labels down. When multiple localities are clumped
    ) +                 # this variable will control how far the labels should 
                        # be placed in the map to avoid clumping the text
  
  # --- MANUAL SCALES for better control over appearance ---
  scale_color_manual(                # We select the colors to be used in Layer 2
    name = "Species Detected",
    values = c("Kimori treeckoi" = "#59a89c",
               "Mizugorou mudkipus" = "#8ec1da",
               "Achamo torchicum" = "#d47264")
  ) +
  scale_shape_manual(                # We select the shapes to be used in Layer 
                                     #2. There are 25 basic shapes availabl in R.
    name = "Species Detected",
    values = c("Kimori treeckoi" = 15,             # Square
               "Mizugorou mudkipus" = 17,        # Triangle 
               "Achamo torchicum" = 18             # Tiny rhombus
               ) 
  ) +
  
  # Add ticks every 10 degrees for the X-axis (Longitude)
  scale_x_continuous(breaks = seq(from = -40, to = 80, by = 10)) +
  
  # Add ticks every 5 degrees for the Y-axis (Latitude)
  scale_y_continuous(breaks = seq(from = 30, to = 80, by = 5)) +
  
  # Layer 4: Set the projection and zoom
  coord_sf(
    crs = crs_lambert,
    xlim = st_bbox(st_transform(st_as_sfc(map_bounds_latlon), crs_lambert))[c("xmin", "xmax")],
    ylim = st_bbox(st_transform(st_as_sfc(map_bounds_latlon), crs_lambert))[c("ymin", "ymax")]
  ) +

  # Layer 4.1: Plain representation without Lambert projection
  # coord_sf(
  #  xlim = map_bounds_latlon[c("xmin", "xmax")],
  #  ylim = map_bounds_latlon[c("ymin", "ymax")],
  #  expand = FALSE # Keep this to prevent extra padding
  # ) +
  
  # Layer 5: Add map aesthetics - scale bar
  annotation_scale(
    location = "bl",            # 'bl' for bottom-left location of the bar. For 
                                # 'top' locations use 't', e.g. 'tl' - top-left
    width_hint = 0.25,          # Adjust bar to occupy n% of the map width
    height = unit(0.15, "cm"),  # Adjust height of the bar
    text_cex = 0.7              # Adjust size of text in the scale bar
  ) +
  
   # Layer 5.1: Add map aesthetics - north arrow
  
  annotation_north_arrow(
    location = "tl",                          # 'tl' places it in the top-left corner
    which_north = "true",                     # Ensures it points to true north
    style = north_arrow_fancy_orienteering    # Chooses a specific visual style
  ) +
  
  # --- CONTROL LEGEND SYMBOL SIZE --- 
  guides(color = guide_legend(override.aes = list(
    size = 4                    # Controls the size of shapes in the legend
  ))) +
  
  # Layer 6: Final theme and labels
  labs(
    title = "Map of Sampled Localities in Europe",         # All of this is quite self-explanatory
    subtitle = "Points colored by species reported",
    x = "Longitude", 
    y = "Latitude"
  ) +
  theme_bw() +
  theme(
    legend.position = "bottom",                 # Controls where to place the legend
    legend.title = element_text(size = 10), 
    legend.text = element_text(size = 8), 
    legend.key.size = unit(0.5, "cm"), 
    legend.key.spacing.x = unit(0.2, "cm")
  )
```

![](README_files/figure-gfm/create-final-map-1.png)<!-- -->

<div class="figure" style="text-align: center">

<img src="Result_image.png" alt="**Fig. 2.** Example of the final map obtained with the methodology explained in this tutorial" width="80%" />
<p class="caption">

**Fig. 2.** Example of the final map obtained with the methodology explained in this tutorial
</p>

</div>

In case you are only interested in a map without any localities
represented in it, simply comment the lines of code comprised within
`Layers 2 & 3`. Additionally, if you prefer a map without a projection,
substitute in the code `Layer 4` for the commented `Layer 4.1`.

------------------------------------------------------------------------

# Final Remarks

The spirit of this tutorial is to facilitate the creation of map figures
for your research . Whether you are preparing a figure for a publication
or simply interested in visualizing spatial distribution data of
species. Needless to say, there are plenty of maps out there and each
one serves a purpose. However, this one will come in handy in most
cases, especially in the field of Biology. And last but not least, Free
Palestine.



