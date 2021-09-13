
# QuickWin tutorial

## Who is this tutorial for

This QuickWin tutorial is for an audience that wants to get a straight to the point tutorial on how to write a tool definition and a workflow in Common Workflow Language (CWL) without knowing details of the specifications.

## What is converted into CWL in this tutorial

The `bash` script to be converted to a CWL script is displayed below.
It takes three URLs to Landsat-8 red, green and blue COGs and uses `gdal_translate` to crop them over an area of interest expressed as a bbox in a given EPSG code. 

```console

red_channel=$1
green_channel=$2
blue_channel=$3
boox="$4"
epsg="$5" 

# epsg default value
[ -z "${epsg}" ] && epsg="EPSG:4326"

cropped=()

for asset_href in "${red_channel}" "${green_channel}" "${blue_channel}" 
do

    in_tif=$( echo "/vsicurl/${asset_href}" || echo ${asset_href} | sed 's/TIF/tif/' ) 
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

done
```

If you have `gdal_translate` on your computer (running Linux or MacOS), this script can be run by saving it in a file called `pan-sharpen.sh` and invoked with:

```console
sh pan-sharpen.sh \
    "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B4.TIF" \
    "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B3.TIF" \
    "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B2.TIF" \
    "13.024,36.69,14.7,38.247" \
    "EPSG:4326"
```

## Recipe

Using a text editor, put the CWL document skeleton below in a file called `translate.cwl`.

```yaml
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id:
  requirements: [] 
  baseCommand: []
  arguments: []
  inputs: {}
  outputs: {}
```

Now set:

- the tool id to `translate` since the CWL document has `$graph` and thus may include several tools 
- the `baseCommand` to `gdal_translate`, which is the command we want CWL to invoke:

```yaml hl_lines="5 8"
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id: translate  
  requirements: []
  baseCommand: 
  - gdal_translate
  arguments: []
  inputs: {}
  outputs: {}
```

Add the `inputs` as type `string`:

- `asset_ref`: the reference to the Landsat-8 COG file 
- `bbox`: the area of interest expressed as a bounding box
- `epsg`: the EPSG code used to express the bounding box coordinates

```yaml hl_lines="11-16"
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id: translate
  requirements: []
  baseCommand: 
  - gdal_translate
  arguments: []
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: {}
```

Add the `arguments` to complete the `gdal_translate` invocation


```yaml hl_lines="9-17"
cwlVersion: v1.0
$graph:
- class: CommandLineTool
  id: translate
  requirements: []
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: {}
```
Notice that CWL expressions are used, e.g.:

```yaml
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
```

And Javascript expressions, e.g.:

```yaml
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
```

Since this CWL document uses javascript expressions to split the `bbox` parameter and set the cropped tif filename, it needs to declare the CWL `InlineJavascriptRequirement` requirement:

```yaml hl_lines="7"
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: {}
```

Add an output of type `File` (the cropped tif) to the `outputs` section:

```yaml hl_lines="28-31"
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

`gdal_translate` will run in a container so set the CWL `DockerRequirement` and the container image URL by using the `dockerPull` field:

```yaml hl_lines="8-9"
cwlVersion: v1.0

$graph:
- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

At this stage, the CWL document is complete ready to run the `gdal_translate` part of the initial shell script.
To do so, prepare the parameters file named `params.yml` with:

```yaml
asset_href: "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B4.TIF"
bbox:  "13.024,36.69,14.7,38.247"
epsg: "EPSG:4326"
```

Run the tool with:

```console
cwltool translate.cwl#translate params.yml
```

The CWL runner takes care of mounting the volumes required for the execution.

The next steps includes adding the `Workflow` section to address the loop on the red, green and blue assets. Start by adding the `Workflow` structure:

```yaml hl_lines="4-11"
cwlVersion: v1.0

$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: {}
  inputs: {}
  outputs: {}
  steps: {}


- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Add the CWL `Workflow` `inputs`:

```yaml hl_lines="10-19"
cwlVersion: v1.0

$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: {}
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: {}
  steps: {}

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Now define the CWL `Workflow` step that will run the `translate` CommandLineTool. For that use the `CommandLineTool` `id` as the `step` `run` value  between double quotes and starting with the hash sign:

```yaml hl_lines="20-23"
$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: {}
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: {}
  steps: 
    node_translate: 
      in:
      out:
      run: "#translate"

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Map the `node_translate` inputs to the `Workflow` inputs. The `asset_ref` input will loop over the `red_channel`, `green_channel` and `blue_channel`. To do so, CWL's `MultipleInputFeatureRequirement` requirement is used and thus added to the `Workflow` requirements: 

```yaml hl_lines="7 23-25"
$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: 
    - class: MultipleInputFeatureRequirement
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: {}
  steps: 
    node_translate: 
      in:
        asset_ref: [red_channel, green_channel, blue_channel]
        bbox: bbox
        epsg: epsg
      out:
      run: "#translate"

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

The loop over the `red_channel`, `green_channel` and `blue_channel` for `asset_href` uses the CWL requirement `ScatterFeatureRequirement` and it defines the `scatter` parameter and method:

```yaml hl_lines="29-30"
$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: 
    - class: MultipleInputFeatureRequirement
    - class: ScatterFeatureRequirement
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: {}
  steps: 
    node_translate: 
      in:
        asset_ref: [red_channel, green_channel, blue_channel]
        bbox: bbox
        epsg: epsg
      out:
      run: "#translate"
      scatter: asset_ref
      scatterMethod: dotproduct 

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Set the `node_translate` step output:

```yaml hl_lines="27-28"
$graph:
- class: Workflow
  id: 
  label: 
  doc:
  requirements: 
    - class: MultipleInputFeatureRequirement
    - class: ScatterFeatureRequirement
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: {}
  steps: 
    node_translate: 
      in:
        asset_href: [red,_channel green_channel, blue_channel]
        bbox: bbox
        epsg: epsg
      out:
      - tif
      run: "#translate"
      scatter: asset_href
      scatterMethod: dotproduct 

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest  
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

And finally the `Workflow` `outputs`:

```yaml hl_lines="21-24"
$graph:
- class: Workflow
  id:
  label: 
  doc:
  requirements: 
    - class: MultipleInputFeatureRequirement
    - class: ScatterFeatureRequirement
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: 
    tifs:
      outputSource:
      - node_translate/tif
      type: File[]
  steps: 
    node_translate: 
      in:
        asset_ref: [red, green, blue]
        bbox: bbox
        epsg: epsg
      out:
      - tif
      run: "#translate"
      scatter: asset_ref
      scatterMethod: dotproduct 

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Set the `Workflow` `id`, `label` and `doc`:

```yaml hl_lines="5-7"
cwlVersion: v1.0

$graph:
- class: Workflow
  id: cropper
  label: this is a label
  doc: this is a description
  requirements: 
    - class: ScatterFeatureRequirement
    - class: MultipleInputFeatureRequirement
  inputs: 
    red_channel: 
      type: string
    green_channel: 
      type: string
    blue_channel: 
      type: string
    bbox: 
      type: string
    epsg: 
      type: string
  outputs: 
    tifs:
      outputSource:
      - node_translate/tif
      type: File[]
  steps: 
    node_translate:
      in:
        asset_href: [red_channel, green_channel, blue_channel]
        bbox: bbox
        epsg: epsg
      out:
      - tif
      run: "#translate"
      scatter: asset_href
      scatterMethod: dotproduct 

- class: CommandLineTool
  id: translate
  requirements: 
    - class: InlineJavascriptRequirement
    - class: DockerRequirement
      dockerPull: docker.io/osgeo/gdal:latest  
  baseCommand: 
  - gdal_translate
  arguments: 
  - -projwin 
  - valueFrom: ${ return inputs.bbox.split(",")[0]; }
  - valueFrom: ${ return inputs.bbox.split(",")[3]; }
  - valueFrom: ${ return inputs.bbox.split(",")[2]; }
  - valueFrom: ${ return inputs.bbox.split(",")[1]; }
  - -projwin_srs
  - $( inputs.epsg )
  - $( inputs.asset_href )
  - valueFrom: ${ return inputs.asset_href.split("/").slice(-1)[0].replace("TIF", "tif"); }
  inputs:
    asset_href:
      type: string
    bbox:
      type: string
    epsg:
      type: string  
  outputs: 
    tif:
      outputBinding:
        glob: '*.tif'
      type: File
```

Create a YAML file called `workflow-params.yml` with:

```yaml
red_channel: "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B4.TIF"
green_channel: "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B3.TIF"
blue_channel: "/vsicurl/https://landsat-pds.s3.amazonaws.com/c1/L8/189/034/LC08_L1TP_189034_20200427_20200509_01_T1/LC08_L1TP_189034_20200427_20200509_01_T1_B2.TIF"
bbox:  "13.024,36.69,14.7,38.247"
epsg: "EPSG:4326"
```

Run the tool with:

```console
cwltool workflow.cwl#cropper workflow-params.yml
```

