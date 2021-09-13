# Shell script to be converted to CWL

The shell script below takes as input:

- `stac_item`: an URL to a STAC Item describing a Landsat-8 acquisition
- `bbox`: the area of interest expressed as a bounding box as `"x_min,y_min,x_max,y_max"`
- `epsg`: the EPSG code used for the bounding box coordinates

to apply the pan-sharpening technique and thus create an RGB composite at 15 metres.

This shell script invokes the executables:

- `curl` and `yq` to get and parse the STAC Item to resolve the red (B04), green (B03) and blue (B02) assets 
- `gdal_translate` to extract the area of interest of each of the red, green, blue assets on the one hand and on the other for the panchromatic band (B06)
- `otbcli_ConcatenateImages` to create a multi-band GeoTIFF file with the red, green, blue cropped bands
- `otbcli_BundleToPerfectSensor` to apply the pan-sharpening technique and generate the higher resolution RGB composite

```bash
--8<--
101/cwl-101/pan-sharpen.sh
--8<--
```

## Identifying the tools 

The shell script invokes four executables:

- `curl` and `yq`
- `gdal_translate`
- `otbcli_ConcatenateImages`
- `otbcli_BundleToPerfectSensor`

### curl and yq

`curl` and `yq` are invoked to resolve the STAC Item assets `B04`, `B03` and `B02`:

```bash hl_lines="3"
for asset in "B4" "B3" "B2" 
do
    asset_href=$( curl -s ${stac_item} | jq .assets.${asset}.href | tr -d '"' )

    in_tif=$( [[ ${asset_href} == http* ]] && echo "/vsicurl/${asset_href}" || echo ${asset_href} | sed 's/TIF/tif/' ) 
    out_tif=$( echo $asset_href | rev | cut -d "/" -f 1 | rev | sed 's/TIF/tif/' )
```

It takes two command line arguments:

- `stac_item`, the URL to the Landsat-8 STAC item
- `asset`, the asset id 

And returns the asset href with the prefix `/vsicurl/` so it can be read by `gdal_translate`.

### gdal_translate

`gdal_translate` is used to read the remote GeoTIFF and save a cropped GeoTIFF.

```bash hl_lines="4-10"
in_tif=$( [[ ${asset_href} == http* ]] && echo "/vsicurl/${asset_href}" || echo ${asset_href} | sed 's/TIF/tif/' ) 
out_tif=$( echo $asset_href | rev | cut -d "/" -f 1 | rev | sed 's/TIF/tif/' )

gdal_translate \
    -projwin \
    $( echo ${bbox} | cut -d "," -f 1 ) \
    $( echo ${bbox} | cut -d "," -f 4 ) \
    $( echo ${bbox} | cut -d "," -f 3 ) \
    $( echo ${bbox} | cut -d "," -f 2 ) \
    -projwin_srs ${epsg} \
    ${in_tif} \
    ${out_tif}

cropped+=($out_tif)
```

It takes as command line arguments: 

- `bbox` whose value is split since `gdal_translate` expects the area of interest expresses as `x_min,y_max,x_max,y_min` while the script gets it as `x_min,y_min,x_max,y_max`.
- `epsg`, the EPSG code used for the bbox coordinates
- `in_tif`, the asset href value prefixed with `/vsicurl`
- `out_tif`, the output name for the cropped GeoTIFF

To produce the cropped GeoTIFF.

### otbcli_ConcatenateImages

Orfeo's Toolbox `otbcli_ConcatenateImages` is used to concatenate the three cropped GeoTIFFs for the red, green and blue channels:

```bash 
otbcli_ConcatenateImages \
    -out \
    ${xs} \
    -il $( for el in ${cropped[@]} ; do echo $el ; done )
```

It takes as command line arguments:

- `${xs}`, the stacked output GeoTIFF
- the list of GeoTIFF to concatenate 

To produce the stacked GeoTIFF.

### otbcli_BundleToPerfectSensor

Orfeo's Toolbox `otbcli_BundleToPerfectSensor` is used to apply the pan-sharpening technique using the stacked GeoTIFF produced by `otbcli_ConcatenateImages` and the panchromatic band cropped by `gdal_translate`

```bash 
otbcli_BundleToPerfectSensor \
    -out pan-sharpen.tif \
    int \
    -inxs ${xs} \
    -inp ${pan}
```

It takes as command line arguments:

- `${xs}`, the stacked GeoTIFF 
- `${pan}`, the cropped pan-chromatic band

To produce the pan-sharpened GeoTIFF.