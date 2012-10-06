---
layout: post
title: "Hillshading with Ordnance Survey Panorama Data"
date: 2012-09-01 15:27
comments: true
categories: [Ordnance Survey, Opendata, Hillshade, Panorama, GDAL, GIS] 
---

This is a guide to creating some nice hillshaded GeoTIFFs using the Ordnance Survey Panorama Opendata. This example will run through a single tile of data — in this case SZ which contains some nice hills plus coastline. Throughout this guide we’ll be using EPSG:27700 as the SRS but obviously, others can be used (for Google Maps et al - EPSG:3857). All data can be downloaded from the [Ordnance Survey website](https://www.ordnancesurvey.co.uk/opendatadownload/products.html).

<!-- more -->

For this recipe, you will need the following:

+ The Vector Map District SZ land data shape file - SZ_Land.shp
+ The OS Panorama datasets for SZ

Firstly, from Panorama, create some initial GeoTiffs from the ESRI ASCII files, for example:

{% codeblock lang:bash %}
gdal_translate -a_srs EPSG:27700 sz88.asc sz88.tiff
{% endcodeblock %}

The following bash script will do the lot in one directory:

{% codeblock lang:bash %}
for i in `ls *asc`; do gdal_translate -a_srs EPSG:27700 $i $i.tiff ; done
{% endcodeblock %}

Now we merge the tiffs into one single image:

{% codeblock lang:bash %}
TIFFS=''
for i in `ls *.tiff`; do TIFFS="$i "$TIFFS ; done
gdal_merge.py -o merged.tiff $TIFFS
{% endcodeblock %}

At this point, you can open up the tiff to see something not very interesting, but you should be able to see stuff. If you open in an image editor, be careful not to save it back as it overwrite the geo information stored in the metadata.

The next step is to do some data resampling ([Creating Hillshaded Tile Overlays](http://alastaira.wordpress.com/2011/07/20/creating-hill-shaded-tile-overlays/))

{% codeblock lang:bash %}
gdalwarp \ 
  -dstnodata 0 \
  -tr 7 7 \
  -r cubicspline \
  -multi -co "TILED=YES" \
  merged.tiff merged.res.tiff
{% endcodeblock %}

Once we’ve created the higher resolution data, we run the hillshade

{% codeblock lang:bash %}
gdaldem hillshade merged.res.tiff merged.res.hs.tiff
{% endcodeblock %}

You can now open the new tiff and see some lovely hillshading, although, we still need to remove the sea from the result. We do this by clipping to the coastline using our shape file ([Create beautiful hillshade maps from digital elevation models with GDAL and Mapnik](http://www.mikejcorey.com/wordpress/2011/02/05/tutorial-create-beautiful-hillshade-maps-from-digital-elevation-models-with-gdal-and-mapnik/)).

First off, we get the extent of the shapefile:

{% codeblock lang:bash %}
ogrinfo -al SZ_Land.shp
{% endcodeblock %}

And using the extent info (for SZ should look like Extent: (400000.000000, 75255.528637) - (499024.062190, 100000.000000)) we trim the tiff:

{% codeblock lang:bash %}
gdal_translate \
  -projwin 400000.000000 100000.000000 499024.062190 75255.528637 \
  merged.res.hs.tiff merged.res.box.tiff
{% endcodeblock %}

Be aware of the order for the -projwin parameter - its not the same as reported by the extent! It requires the NW -> SE points as opposed to SW -> NE.

And finally, the alpha and cropping:

{% codeblock lang:bash %}
gdalwarp \ 
  -dstalpha \
  -srcnodata 0 \
  -dstnodata 0 \
  -cutline SZ_Land.shp \
  -crop_to_cutline \
  merged.res.box.tiff merged.res.hs.final.tiff
{% endcodeblock %}
