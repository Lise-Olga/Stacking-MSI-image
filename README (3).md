# Stacking Multispectral Bands into a Single Image

*A step-by-step guide for GIS and remote sensing users, with Python and rasterio*

Multispectral remote sensing images often come as one file per band. Many analyses, and all deep learning models, need these bands in a single file, in a fixed order. This guide follows the five parts of the notebook (Stacking-MSI-image
), in the same order.

The method works for any multispectral image with one raster per band, from a satellite or a drone. The example results come from a MicaSense Altum drone flight over a wheat field (blue, green, red, red edge and near infrared). Notes in each step explain what changes for satellite images.

> **ArcGIS equivalent:** the whole process does the same job as the Composite Bands tool, with a few checks before and after.

## What you need

- **Libraries:** rasterio reads and writes georeferenced rasters (GeoTIFF). It is built on GDAL, the same library used by many GIS programs. Install it once with `pip install rasterio`. The notebook also uses numpy (calculations), glob and os (finding files) and matplotlib (graphs).

- **Data:** one single-band reflectance raster per band, all from the same scene or flight. The notebook looks for `.tif` files; Sentinel-2 bands come as `.jp2` files, which rasterio also reads, so only the extension in step 1 needs to change.

- **The notebook.** To use it with your own data, change the folder path (step 1), the file names, band order and number of bands (step 3), the path of the stacked image (step 4) and the wavelengths of your sensor (step 4).

## 1. Folder path

You tell the notebook which folder holds the reflectance maps. It then lists the files. In the example, each band has a `.tif` of about 93 MB, with a `.tfw` and a `.prj` file. Only the `.tif` files are used, because they already contain the georeferencing.

## 2. Check coordinates

Before stacking, all bands must match exactly: same number of rows and columns, same coordinate system and same pixel size. If one band is different, the pixels will not line up and the stacked image will be wrong, without any error message.

The notebook opens each band, without changing anything, and prints its properties. In the example, it finds five files, and all five match:

| Property   | Value            | Meaning                                      |
|----------------|----------------------|--------------------------------------------------|
| Size           | 4,000 × 7,231 pixels | Same for every band                              |
| Bands per file | 1                    | As expected                                      |
| Data type      | `float32`              | Decimal numbers, normal for reflectance (0 to 1) |
| CRS            | `EPSG:32614`           | WGS 84 / UTM zone 14N, in meters                 |
| Pixel size     | 0.0226 m             | About 2.3 cm on the ground                       |

If a value were different, you would first need to resample or reproject that band (in ArcGIS: Resample or Project Raster). Here, everything lines up, so the bands can be stacked.

**Satellite note.** This check matters a lot for Sentinel-2, whose bands have different pixel sizes: 10 m for blue, green, red and NIR, and 20 m for the red edge and SWIR bands. Resample them to the same pixel size before stacking. Landsat 8 and 9 multispectral bands are all 30 m.

## 3. Stack the bands

The notebook creates one multiband image, with the bands sorted by wavelength. It first sets the order with a list, because the alphabetical order of the file names would put NIR before red. Each sensor and each provider names its files differently, so this list is the main part to adapt.

Next, it copies the profile of the first band (size, coordinate system, georeferencing, data type, NoData value) and changes three things: the number of bands (five in the example), LZW compression (a smaller file with no data loss) and 512 × 512 pixel tiles (faster display in GIS software).

Finally, it writes each band in its place, gives it a name and prints which file went into which band. In the example, the result is a 5-band file:

| Band | Name | Center wavelength (Altum) |
|----------|----------|-------------------------------|
| 1        | Blue     | 475 nm                        |
| 2        | Green    | 560 nm                        |
| 3        | Red      | 668 nm                        |
| 4        | RedEdge  | 717 nm                        |
| 5        | NIR      | 842 nm                        |

Other sensors have other wavelengths and often a different number of bands (Sentinel-2 has 13, Landsat 8 and 9 have 11). Choose the bands you need and use the wavelengths from the sensor documentation.

> **ArcGIS equivalent:** the Composite Bands tool, with the input rasters sorted by wavelength.

Every later step depends on this order. To compute NDVI, you need to know which band is red and which is NIR (bands 3 and 5 in the example). A deep learning model also learns from the bands in a fixed order, so every image you give it must be stacked the same way. Because the names are saved in the file, ArcGIS Pro shows Blue, Green, Red, RedEdge and NIR instead of Band_1 to Band_5.

## 4. Spectral profile

The image is stacked; now you check that the values make sense for the surface. The spectral profile shows the mean reflectance of the image in each band. In the example, the image covers one wheat field, so the curve should look like vegetation. For a satellite scene with many land covers, the mean mixes them all: it is better to compute the profile inside a polygon of one class (a field, a lake, a city).

To save time, the notebook reads the image 10 times smaller in each direction. It keeps only the pixels between 0 and 1, which removes the NoData area around the field. In the example, 245,103 pixels remain. It computes the mean and standard deviation of each band, plots the curve against the wavelengths and saves it as a PNG.

> **ArcGIS equivalent:** the Spectral Profile chart in ArcGIS Pro (Image Analyst), here for the whole field.

The curve has the typical shape of vegetation:

| Band | Mean | Std. dev. | What it shows                                      |
|----------|----------|---------------|--------------------------------------------------------|
| Blue     | 0.024    | 0.010         | Absorbed by chlorophyll: low                           |
| Green    | 0.057    | 0.018         | Reflected a little more: this is why plants look green |
| Red      | 0.059    | 0.041         | Absorbed by chlorophyll: should be low                 |
| Red edge | 0.111    | 0.034         | The sharp rise between red and NIR                     |
| NIR      | 0.439    | 0.144         | Strongly reflected by the leaf structure: high         |

One point needs attention. In healthy green vegetation, red is usually lower than green. Here, red (0.059) is almost equal to green (0.057), and its standard deviation is high (0.041). The field probably contains pixels that are not green vegetation, such as bare soil between rows or yellowing plants. This should be checked on the map. It matters for a disease study, where leaf yellowing is part of the signal.

## 5. Reflectance

The last check: reflectance should be between 0 and 1. The notebook reads the image the same way as in step 4 and prints the NoData value of the file (-10,000 in the example). Then, for each band, it prints the minimum, maximum, median, 2nd and 98th percentiles, and the percentage of pixels below 0 or above 1. This time, the NoData pixels are not removed.

> **ArcGIS equivalent:** Statistics and Histogram in the layer properties.

| Band | Median | 98th percentile | Maximum | Pixels < 0 |
|----------|------------|---------------------|-------------|-----------------|
| Blue     | 0.020      | 0.045               | 0.082       | 15.2%           |
| Green    | 0.051      | 0.095               | 0.148       | 15.2%           |
| Red      | 0.034      | 0.143               | 0.385       | 15.1%           |
| Red edge | 0.098      | 0.182               | 0.292       | 15.1%           |
| NIR      | 0.377      | 0.745               | 0.931       | 15.0%           |

About 15% of the pixels equal -10,000 in each band. This is the area outside the field: the image is a rectangle, but the field is not. It is also why the minimum and the 2nd percentile equal -10,000. Apart from that, no pixel is above 1, the maximums are realistic and the medians follow the same vegetation shape as the spectral profile.

**Satellite note.** Many products store reflectance as integers, with another NoData value. Landsat Collection 2 Level-2: reflectance = DN × 0.0000275 − 0.2, with NoData = 0. Sentinel-2 Level-2A (since 2022): reflectance = (DN − 1000) / 10,000, with NoData = 0. If your values are not between 0 and 1, convert them before step 4, or the filter in step 4 will remove almost every pixel.

> **Important:** the NoData value is saved in the stacked image, but not all software reads it. Before computing an index or training a model, always remove or mask the NoData pixels. A single NoData pixel can distort a mean, an NDVI value or a normalization. In rasterio, the `masked=True` option masks these pixels when reading.

The stacked image is now ready for vegetation indices (NDVI, NDRE) or to be cut into small tiles for a deep learning model.

## Author

Lise Olga Makonga, M.S. student in Geography and Geospatial Sciences.
