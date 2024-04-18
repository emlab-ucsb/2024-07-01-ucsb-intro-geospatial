---
title: Reproject Raster Data
teaching: 40
exercises: 20
source: Rmd
---



::::::::::::::::::::::::::::::::::::::: objectives

- Reproject a raster in R.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I work with raster data sets that are in different projections?

::::::::::::::::::::::::::::::::::::::::::::::::::

Sometimes we encounter raster datasets that do not "line up" when plotted or
analyzed. Rasters that don't line up are most often in different Coordinate
Reference Systems (CRS). This episode explains how to deal with rasters in 
different, known CRSs. It will walk though reprojecting rasters in R using 
the `project()` function in the `terra` package.

Let's load the packages we'll need for this lesson: 


```r
library(terra)
library(ggplot2)
library(dplyr)
```

## Raster Projection in R

In the previous episode, we learned how to layer a raster file on top of a 
hillshade for a nice looking basemap. In that episode, all of our data were
in the same CRS. What happens when things don't line up?

For this episode, we will be working with the Harvard Forest Digital Terrain
Model data. This differs from the surface model data we've been working with so
far in that the digital surface model (DSM) includes the tops of trees, while
the digital terrain model (DTM) shows the ground level.

Here, we will create a map of the Harvard Forest 
Digital Terrain Model (`DTM_HARV`) draped or layered on top of the hillshade 
(`DTM_hill_HARV`).
The hillshade layer maps the terrain using light and shadow to create a 
3D-looking image, based on a hypothetical illumination of the ground level.

![](fig/dc-spatial-raster/lidarTree-height.png){alt='Source: National Ecological Observatory Network (NEON).'}

First, we need to import the DTM and DTM hillshade data.


```r
DTM_HARV <- 
    rast("data/NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_dtmCrop.tif")

DTM_hill_HARV <- 
    rast("data/NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_DTMhill_WGS84.tif")
```

Next, we will convert each of these datasets to a dataframe for
plotting with `ggplot`.


```r
DTM_HARV_df <- as.data.frame(DTM_HARV, xy = TRUE)

DTM_hill_HARV_df <- as.data.frame(DTM_hill_HARV, xy = TRUE)
```

Now we can create a map of the DTM layered over the hillshade. Note that we use
`scale_fill_gradientn()` here to specify our color scale. This allows for us to
easily create a color ramp for the data we want to display.


```r
ggplot() +
     geom_raster(data = DTM_HARV_df , 
                 mapping = aes(x = x, y = y, 
                  fill = HARV_dtmCrop)) + 
     geom_raster(data = DTM_hill_HARV_df, 
                 mapping = aes(x = x, y = y, 
                   alpha = HARV_DTMhill_WGS84)) +
     scale_fill_gradientn(name = "Elevation", colors = terrain.colors(10)) + 
     coord_quickmap()
```

<img src="fig/11-raster-reproject-in-r-rendered-plot-empty-1.png" style="display: block; margin: auto;" />

Our results are curious - neither the Digital Terrain Model (`DTM_HARV_df`)
nor the DTM Hillshade (`DTM_hill_HARV_df`) plotted.
Let's try to plot the DTM on its own to make sure there are data there.


```r
ggplot() +
  geom_raster(data = DTM_HARV_df,
      mapping = aes(x = x, y = y, fill = HARV_dtmCrop)) +
  scale_fill_gradientn(name = "Elevation", colors = terrain.colors(10)) + 
  coord_quickmap()
```

<img src="fig/11-raster-reproject-in-r-rendered-plot-DTM-1.png" style="display: block; margin: auto;" />

Our DTM seems to contain data and plots just fine.

Next we plot the DTM Hillshade on its own to see whether everything is OK.


```r
ggplot() + 
  geom_raster(data = DTM_hill_HARV_df,
              mapping = aes(x = x, y = y,
                            alpha = HARV_DTMhill_WGS84)) + 
  coord_quickmap()
```

<img src="fig/11-raster-reproject-in-r-rendered-plot-DTM-hill-1.png" style="display: block; margin: auto;" />

If we look at the axes, we can see that the projections of the two rasters are 
different.
When this is the case, `ggplot` won't render the image. It won't even throw an 
error message to tell you something has gone wrong. We can look at Coordinate 
Reference Systems (CRSs) of the DTM and the hillshade data to see how they 
differ.


```r
crs(DTM_HARV, proj = TRUE)
```

```{.output}
[1] "+proj=utm +zone=18 +datum=WGS84 +units=m +no_defs"
```

```r
crs(DTM_hill_HARV, proj = TRUE)
```

```{.output}
[1] "+proj=longlat +datum=WGS84 +no_defs"
```

`DTM_HARV` is in the UTM projection, with units of meters.
`DTM_hill_HARV` is in
`Geographic WGS84` - which is represented by latitude and longitude values.

Because the two rasters are in different CRSs, they don't line up when plotted
in R. We need to reproject (or change the projection of) `DTM_hill_HARV` into 
the UTM CRS. Alternatively, we could reproject `DTM_HARV` into WGS84.

## Reproject Rasters

We can use the `project()` function to reproject a raster into a new CRS.
Keep in mind that reprojection only works when you first have a defined CRS
for the raster object that you want to reproject. It cannot be used if no
CRS is defined. Lucky for us, the `DTM_hill_HARV` has a defined CRS.

:::::::::::::::::::::::::::::::::::::::::  callout

## Data Tip

When we reproject a raster, we move it from one "grid" to another. Thus, we are 
modifying the data! Keep this in mind as we work with raster data.


::::::::::::::::::::::::::::::::::::::::::::::::::

To use the `project()` function, we need to define two things:

1. the object we want to reproject and
2. the CRS that we want to reproject it to.

The syntax is `project(x = RasterObject, y = crs)`

We want the CRS of our hillshade to match the `DTM_HARV` raster. One option 
would be to just use the CRS of `DTM_HARV` in the `project()` function, but
this will cause other issues later down the line because the resolutions of 
`DTM_HARV` and `DTM_hill_HARV` are different. That's okay, because we can specify
the resolution of the output file in the `project()` function using `res`. 

That's a little complicated for our needs though. In our case, the most 
effective way to reproject the dataset is to use the `DTM_HARV` layer as a
template for the reprojection. We can specify that by using the syntax
`project(x = RasterObject, y = TemplateRasterObject)`. 

Note that we are using the `project()` function on the raster object,
not the `data.frame()` we use for plotting with `ggplot`.



```r
DTM_hill_UTMZ18N_HARV <- project(x = DTM_hill_HARV,
                                 y = DTM_HARV)
```

For plotting with `ggplot()`, we will need to create a dataframe from our newly 
reprojected raster.


```r
DTM_hill_HARV_2_df <- as.data.frame(DTM_hill_UTMZ18N_HARV, xy = TRUE)
```

We can now create a plot of this data.


```r
ggplot() +
     geom_raster(data = DTM_HARV_df , 
                 mapping = aes(x = x, y = y, 
                               fill = HARV_dtmCrop)) + 
     geom_raster(data = DTM_hill_HARV_2_df, 
                 mapping = aes(x = x, y = y, 
                               alpha = HARV_DTMhill_WGS84)) +
     scale_fill_gradientn(name = "Elevation", colors = terrain.colors(10)) + 
     coord_quickmap()
```

<img src="fig/11-raster-reproject-in-r-rendered-plot-projected-raster-1.png" style="display: block; margin: auto;" />

We have now successfully draped the Digital Terrain Model on top of our
hillshade to produce a nice looking, textured map!

:::::::::::::::::::::::::::::::::::::::  challenge

## Challenge 1: Reproject, then Plot a Digital Terrain Model

Create a map of the
[San Joaquin Experimental Range](https://www.neonscience.org/field-sites/field-sites-map/SJER)
field site using the `SJER_DSMhill_WGS84.tif` and `SJER_dsmCrop.tif` files.

Reproject the data as necessary to make things line up!

:::::::::::::::  solution

## Solution


```r
# import DSM
DSM_SJER <- 
    rast("data/NEON-DS-Airborne-Remote-Sensing/SJER/DSM/SJER_dsmCrop.tif")
# import DSM hillshade
DSM_hill_SJER_WGS <-
    rast("data/NEON-DS-Airborne-Remote-Sensing/SJER/DSM/SJER_DSMhill_WGS84.tif")

# reproject raster
DSM_hill_UTMZ18N_SJER <- project(x = DSM_hill_SJER_WGS,
                                 y = DSM_SJER)

# convert to data.frames
DSM_SJER_df <- as.data.frame(DSM_SJER, xy = TRUE)

DSM_hill_SJER_df <- as.data.frame(DSM_hill_UTMZ18N_SJER, xy = TRUE)

ggplot() +
     geom_raster(data = DSM_hill_SJER_df, 
                 mapping = aes(x = x, y = y, 
                               alpha = SJER_DSMhill_WGS84)) +
     geom_raster(data = DSM_SJER_df, 
             mapping = aes(x = x, y = y, 
                           fill = SJER_dsmCrop,
                           alpha=0.8)) + 
     scale_fill_gradientn(name = "Elevation", colors = terrain.colors(10)) + 
     coord_quickmap()
```

<img src="fig/11-raster-reproject-in-r-rendered-challenge-code-reprojection-1.png" style="display: block; margin: auto;" />

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::



:::::::::::::::::::::::::::::::::::::::: keypoints

- In order to plot two raster data sets together, they must be in the same CRS.
- Use the `project()` function to convert between CRSs.

::::::::::::::::::::::::::::::::::::::::::::::::::


