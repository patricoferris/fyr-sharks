# Uncertainty in Code <uncertainty-in-code>

Uncertainty in computational, ecological methodologies can arise from the environment.
This includes the installed libraries and packages on the machine running any analysis.

For example, consider calculating the _Terrain Ruggedness Index_ (TRI) using GDAL. First we grab
two versions of GDAL. The first is **v3.2**.

```shark-build:v3.2
((from "osgeo/gdal:ubuntu-small-3.2.0")
 (run (shell "mkdir /data")))
```

And the second is **v3.3**.

```shark-build:v3.3
((from "osgeo/gdal:ubuntu-small-3.3.0")
 (run (shell "mkdir /data")))
```

## The dataset

We will use the SRTM dataset as an example of what can go wrong. 

```shark-run:v3.2
$ curl -s https://srtm.csi.cgiar.org/wp-content/uploads/files/srtm_5x5/TIFF/srtm_34_11.zip > /data/srtm_34_11.zip
```

We can then unzip and copy the tile out of the downloaded elevation map. 

```shark-run:v3.2
$ unzip /data/srtm_34_11.zip -d /data/srtm_34_11-raw
$ sha256sum /data/srtm_34_11-raw/srtm_34_11.tif
```

## Preprocessing

Before doing the analysis, we need to warp the raw image into a suitable CRS.

```shark-run:v3.3
$ gdalwarp -s_srs "EPSG:4326" -t_srs '+proj=utm +zone=29 +datum=WGS84 +units=m +no_defs' -of GTIFF /data/srtm_34_11-raw/srtm_34_11.tif /data/srtm_34_11.tif
```

## Calculate TRI

There is a command line tool that is packaged with GDAL to calculate TRI.

We can do this with tool packaged with GDAL **v3.2**.

```shark-run:v3.2
$ gdaldem tri -q /data/srtm_34_11.tif /data/srtm_34_11-v3.2.tif
```

And then again for the same tool package with GDAL **v3.3**.

```shark-run:v3.3
$ gdaldem tri -q /data/srtm_34_11.tif /data/srtm_34_11-v3.3.tif
```

First we perform a test where the difference between two rasters should be nothing.

```shark-run:v3.2
$ gdal_calc.py --quiet --cal="A - B" -A /data/srtm_34_11-v3.2.tif -B /data/srtm_34_11-v3.2.tif --outfile=/data/srtm-tri-no-diff.tif
$ gdalinfo -stats /data/srtm-tri-no-diff.tif | sed -n '/Band 1 Block/,$p'
```

No we do the same but for the two TRIs generate with GDAL **v3.2** and **v3.3**.

```shark-run:v3.2
$ gdal_calc.py --quiet --cal="A - B" -A /data/srtm_34_11-v3.2.tif -B /data/srtm_34_11-v3.3.tif --outfile=/data/srtm-tri-diff.tif
$ gdalinfo -stats /data/srtm-tri-diff.tif | sed -n '/Band 1 Block/,$p'
```

The diff image can then be published locally by [Shark](https://github.com/quantifyearth/shark).

```shark-publish
/data/srtm-tri-diff.tif
```


