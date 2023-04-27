# Satellite-image-from-google-earth-engine
This code will describe about how to create a polygon on map and which is used as area of interest (Aoi)
for getting the satellite images of that area with different bands files saved into the output directory.

pip install ee							#for working with google earth engine

import ee
ee.Authenticate()
ee.Initialize()

To authorize access needed by Earth Engine, open the following URL in a web browser and follow the instructions:


The authorization workflow will generate a code, which you should paste in the box below.

Enter verification code: *************************************************
Successfully saved authorization token.


import geopandas as gpd
from shapely.geometry import Polygon
from ipyleaflet import Map, DrawControl

# Open a map and add a drawing control to draw a polygon
m = Map(center=(37.5, -122.0), zoom=10)
draw_control = DrawControl(marker={'opacity': 0.5}, polygon={'opacity': 0.5})
m.add_control(draw_control)
m

Map(center=[37.5, -122.0], controls=(ZoomControl(options=['position', 'zoom_in_text', 'zoom_in_title', 'zoom_o…

# Get the coordinates of the polygon
aoi_polygon = draw_control.last_draw['geometry']['coordinates'][0]
print("The coordinates of the polygon are:")
print(aoi_polygon)

# Crop the GeoDataFrame object using the polygon
world = gpd.read_file(gpd.datasets.get_path("naturalearth_lowres"))
world = world.set_index("iso_a3")
aoi1 = world[world.intersects(Polygon(aoi_polygon))]
aoi1 = aoi1.intersection(Polygon(aoi_polygon))

# Display the cropped GeoDataFrame object
aoi1.plot(figsize=(5,5),edgecolor='black', color='orange')

The coordinates of the polygon are:
[[-121.096661, 37.491684], [-121.094945, 37.453265], [-121.012398, 37.456672], [-121.014286, 37.495226], [-121.096661, 37.491684]]
<AxesSubplot:>

aoi = ee.Geometry.Polygon(aoi_polygon)
aoi
ee.Geometry({
  "functionInvocationValue": {
    "functionName": "GeometryConstructors.Polygon",
    "arguments": {
      "coordinates": {
        "constantValue": [
          [
            [
              -121.096661,
              37.491684
            ],
            [
              -121.094945,
              37.453265
            ],
            [
              -121.012398,
              37.456672
            ],
            [
              -121.014286,
              37.495226
            ],
            [
              -121.096661,
              37.491684
            ]
          ]
        ]
      },
      "evenOdd": {
        "constantValue": true
      }
    }
  }
})
collection = ee.ImageCollection("COPERNICUS/S2_SR")\
                .filterDate('2022-10-30', '2022-12-31')\
                .filterBounds(aoi)\
                .filterMetadata('CLOUDY_PIXEL_PERCENTAGE', 'less_than', 20)
collection
<ee.imagecollection.ImageCollection at 0x25df0913a30>
clipped_collection = collection.map(lambda image:image.clip(aoi))
clipped_collection
<ee.imagecollection.ImageCollection at 0x25dec245580>
image_url = clipped_collection.first().getDownloadURL({
    'name':'landsat_image',
    'scale':20,
    'crs':'EPSG:4326',
    'region': aoi.getInfo()["coordinates"]
})

# import os
# import urllib.request

# output_folder = "output" # specify the name of the output folder
# if not os.path.exists(output_folder):
#     os.makedirs(output_folder)

# urllib.request.urlretrieve(image_url, os.path.join(output_folder, 'landsat_image.tif'))
('output\\landsat_image.tif', <http.client.HTTPMessage at 0x179193e02e0>)
import os
import urllib.request

output_folder = "output" # specify the name of the output folder
if not os.path.exists(output_folder):
    os.makedirs(output_folder)

# download the zip file containing all bands
zip_path = os.path.join(output_folder, "landsat_image.zip")
urllib.request.urlretrieve(image_url, zip_path)

# extract all the band files from the zip file
import zipfile
with zipfile.ZipFile(zip_path, 'r') as zip_ref:
    zip_ref.extractall(output_folder)

import rasterio
import matplotlib.pyplot as plt
import numpy as np

# Open each band as a separate rasterio dataset
with rasterio.open('output/landsat_image.B2.tif') as b2, \
     rasterio.open('output/landsat_image.B3.tif') as b3, \
     rasterio.open('output/landsat_image.B4.tif') as b4, \
     rasterio.open('output/landsat_image.B5.tif') as b5, \
     rasterio.open('output/landsat_image.B8A.tif') as b8A:
    
    # Read the raster data as numpy arrays
    b2_data = b2.read(1)
    b3_data = b3.read(1)
    b4_data = b4.read(1)
    b5_data = b5.read(1)
    b8A_data = b8A.read(1)
    
    # Rescale the pixel values for each band
    b8A_data_scaled = (b8A_data - np.min(b8A_data)) / (np.max(b8A_data) - np.min(b8A_data))
    b5_data_scaled = (b5_data - np.min(b5_data)) / (np.max(b5_data) - np.min(b5_data))
    b4_data_scaled = (b4_data - np.min(b4_data)) / (np.max(b4_data) - np.min(b4_data))
    b3_data_scaled = (b3_data - np.min(b3_data)) / (np.max(b3_data) - np.min(b3_data))
    b2_data_scaled = (b2_data - np.min(b2_data)) / (np.max(b2_data) - np.min(b2_data))

    # Combine the rescaled bands into an RGB image
    rgb_image = np.dstack((b5_data_scaled, b4_data_scaled, b3_data_scaled))

    # Display the RGB image
    plt.imshow(rgb_image, vmin=0, vmax=1)
    plt.show()
#####################################################################################################################
    # Combine the rescaled bands into an flase image
    false_image = np.dstack((b8A_data_scaled, b4_data_scaled, b2_data_scaled))

    # Display the RGB image
    plt.imshow(false_image, vmin=0, vmax=1)
    plt.show()
    

    # Plot each band separately
    fig, (ax1, ax2, ax3, ax4,ax5) = plt.subplots(ncols=5, figsize=(15,15))
    ax1.set_title('Band 2')
    ax1.imshow(b2_data_scaled,vmin=0, vmax=1, cmap='Blues')
    ax2.set_title('Band 3')
    ax2.imshow(b3_data_scaled,vmin=0, vmax=1, cmap='Greens')
    ax3.set_title('Band 4')
    ax3.imshow(b4_data_scaled,vmin=0, vmax=1, cmap='Reds')
    ax4.set_title('Band 5')
    ax4.imshow(b5_data_scaled,vmin=0, vmax=1, cmap='RdYlGn')
    ax5.set_title('Band 8A')
    ax5.imshow(b8A_data_scaled,vmin=0, vmax=1, cmap='RdYlBu')
    plt.show()



 
