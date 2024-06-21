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

```shark-run:v3.2:bdcc35b366eb94eb09247a926a6f7d742645115161c16a7b2d0f87651d3ef209
curl -s https://srtm.csi.cgiar.org/wp-content/uploads/files/srtm_5x5/TIFF/srtm_34_11.zip > /data/srtm_34_11.zip
```

We can then unzip and copy the tile out of the downloaded elevation map.

```shark-run:v3.2:ccf6bf2161b7495236ba8aa8c78f9fcdb844b551d06d33f525573d30ae30e73c
unzip /shark/bdcc35b366eb94eb09247a926a6f7d742645115161c16a7b2d0f87651d3ef209/srtm_34_11.zip -d /data/srtm_34_11-raw
Archive:  /shark/bdcc35b366eb94eb09247a926a6f7d742645115161c16a7b2d0f87651d3ef209/srtm_34_11.zip
  inflating: /data/srtm_34_11-raw/readme.txt  
  inflating: /data/srtm_34_11-raw/srtm_34_11.hdr  
  inflating: /data/srtm_34_11-raw/srtm_34_11.tfw  
  inflating: /data/srtm_34_11-raw/srtm_34_11.tif  

sha256sum /data/srtm_34_11-raw/srtm_34_11.tif
c512ed327e814864a56af290d86a93243d26d6098c6968b6f43059cc7c68dc5e  /data/srtm_34_11-raw/srtm_34_11.tif

```

## Preprocessing

Before doing the analysis, we need to warp the raw image into a suitable CRS.

```shark-run:v3.3:67bd64211719d28e4d801527fb7be7f0ee39d2ae6e5caf49ea2cff54314415d5
gdalwarp -s_srs "EPSG:4326" -t_srs '+proj=utm +zone=29 +datum=WGS84 +units=m +no_defs' -of GTIFF /shark/ccf6bf2161b7495236ba8aa8c78f9fcdb844b551d06d33f525573d30ae30e73c/srtm_34_11-raw/srtm_34_11.tif /data/srtm_34_11.tif
Creating output file that is 6037P x 6058L.
Processing /shark/ccf6bf2161b7495236ba8aa8c78f9fcdb844b551d06d33f525573d30ae30e73c/srtm_34_11-raw/srtm_34_11.tif [1/1] : 0Using internal nodata values (e.g. -32768) for image /shark/ccf6bf2161b7495236ba8aa8c78f9fcdb844b551d06d33f525573d30ae30e73c/srtm_34_11-raw/srtm_34_11.tif.
Copying nodata values from source /shark/ccf6bf2161b7495236ba8aa8c78f9fcdb844b551d06d33f525573d30ae30e73c/srtm_34_11-raw/srtm_34_11.tif to destination /data/srtm_34_11.tif.
...10...20...30...40...50...60...70...80...90...100 - done.

```

## Calculate TRI

There is a command line tool that is packaged with GDAL to calculate TRI.

We can do this with tool packaged with GDAL **v3.2**.

```shark-run:v3.2:1397a726f68551fc75e4b14b1454782f698e9c7b3c4bb889558040372de43fd3
gdaldem tri -q /shark/67bd64211719d28e4d801527fb7be7f0ee39d2ae6e5caf49ea2cff54314415d5/srtm_34_11.tif /data/srtm_34_11-v3.2.tif
```

And then again for the same tool package with GDAL **v3.3**.

```shark-run:v3.3:4c12ad4f4d6d47ef5e8922b63887161e18604d62464e081ce6290804e8934d2e
gdaldem tri -q /shark/67bd64211719d28e4d801527fb7be7f0ee39d2ae6e5caf49ea2cff54314415d5/srtm_34_11.tif /data/srtm_34_11-v3.3.tif
```

First we perform a test where the difference between two rasters should be nothing.

```shark-run:v3.2:fad67d1e91fe610853d699fc46fa2a346c0dcf8ab62b1d4401ea2feb7ab71368
gdal_calc.py --quiet --cal="A - B" -A /shark/1397a726f68551fc75e4b14b1454782f698e9c7b3c4bb889558040372de43fd3/srtm_34_11-v3.2.tif -B /shark/1397a726f68551fc75e4b14b1454782f698e9c7b3c4bb889558040372de43fd3/srtm_34_11-v3.2.tif --outfile=/data/srtm-tri-no-diff.tif
gdalinfo -stats /data/srtm-tri-no-diff.tif | sed -n '/Band 1 Block/,$p'
Band 1 Block=6037x1 Type=Float32, ColorInterp=Gray
  Minimum=0.000, Maximum=0.000, Mean=0.000, StdDev=0.000
  NoData Value=3.40282346600000016e+38
  Metadata:
    STATISTICS_MAXIMUM=0
    STATISTICS_MEAN=0
    STATISTICS_MINIMUM=0
    STATISTICS_STDDEV=0
    STATISTICS_VALID_PERCENT=39.57

```

No we do the same but for the two TRIs generate with GDAL **v3.2** and **v3.3**.

```shark-run:v3.2:3dbbd56760c5168b0b1d0594b09b299cc99934453eb1b05bf207233c547f2825
gdal_calc.py --quiet --cal="A - B" -A /shark/1397a726f68551fc75e4b14b1454782f698e9c7b3c4bb889558040372de43fd3/srtm_34_11-v3.2.tif -B /shark/4c12ad4f4d6d47ef5e8922b63887161e18604d62464e081ce6290804e8934d2e/srtm_34_11-v3.3.tif --outfile=/data/srtm-tri-diff.tif
gdalinfo -stats /data/srtm-tri-diff.tif | sed -n '/Band 1 Block/,$p'
Band 1 Block=6037x1 Type=Float32, ColorInterp=Gray
  Minimum=-672.854, Maximum=0.000, Mean=-15.297, StdDev=14.790
  NoData Value=3.40282346600000016e+38
  Metadata:
    STATISTICS_MAXIMUM=0
    STATISTICS_MEAN=-15.297075732539
    STATISTICS_MINIMUM=-672.85357666016
    STATISTICS_STDDEV=14.790352827817
    STATISTICS_VALID_PERCENT=39.57

```

The diff image can then be published locally by [Shark](https://github.com/quantifyearth/shark).

```shark-publish
/obuilder-zfs/result/3dbbd56760c5168b0b1d0594b09b299cc99934453eb1b05bf207233c547f2825/.zfs/snapshot/snap/rootfs/data/srtm-tri-diff.tif
```


