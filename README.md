# Urban Green Space Monitoring using Sentinel-2

This project focuses on monitoring urban green spaces in Berlin, Germany, using Sentinel-2 satellite imagery and Google Earth Engine (GEE). The project calculates the Normalized Difference Vegetation Index (NDVI) to identify and quantify urban green areas.

## Project Overview

Urban green spaces are crucial for the environmental health and quality of life in cities. This project aims to monitor and analyze the extent of green spaces in Berlin, Germany, during the peak summer season (July) using Sentinel-2 imagery.

## Data and Methodology

- **Data Source**: Sentinel-2 imagery from Google Earth Engine
- **Study Area**: Berlin, Germany
- **Time Period**: July 2023
- **Index Used**: Normalized Difference Vegetation Index (NDVI)
- **Tools**: Google Earth Engine

### NDVI Calculation

NDVI is calculated using the following formula:
[ NDVI = (NIR - Red)/(NIR + Red) ]

Where:
- **NIR** is the Near Infrared band (B8)
- **Red** is the Red band (B4)

### Steps

1. **Load Sentinel-2 Data**: Filter the data for Berlin and the month of July 2023.
2. **Calculate NDVI**: Compute the NDVI for the filtered data.
3. **Composite Image**: Create a median composite of NDVI values.
4. **Visualization**: Define visualization parameters and display the NDVI on the map.
5. **Legend and Histogram**: Add a legend and a histogram of NDVI distribution.
6. **Urban Green Area Extraction**: Extract urban green areas based on an NDVI threshold.
7. **Area Calculation**: Calculate the total area of urban green spaces.

## Installation

To run this project, you need access to Google Earth Engine. Follow these steps:

1. Sign up for a Google Earth Engine account [here](https://earthengine.google.com/).
2. Go to the Google Earth Engine Code Editor [here](https://code.earthengine.google.com/).
3. Copy and paste the provided script into the Code Editor.

## Usage

Here's the GEE script for the project:

```javascript
// Define the study area: Berlin, Germany
var berlin = ee.Geometry.Polygon(
  [[[13.0884, 52.6755],
    [13.7611, 52.6755],
    [13.7611, 52.3383],
    [13.0884, 52.3383]]]
);

// Load Sentinel-2 data
var sentinel2 = ee.ImageCollection('COPERNICUS/S2')
  .filterBounds(berlin)
  .filterDate('2023-07-01', '2023-07-31') // Filter for July
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20)); // Filter by cloud coverage

// Compute NDVI
var ndvi = sentinel2.map(function(image) {
  var ndvi = image.normalizedDifference(['B8', 'B4']).rename('NDVI');
  return image.addBands(ndvi);
});

// Select the median NDVI composite
var ndviComposite = ndvi.select('NDVI').median();

// Define visualization parameters with adjusted green value
var ndviParams = {
  min: 0,
  max: 1,
  palette: ['red', 'yellow', 'lightgreen', 'green']
};

// Center the map on Berlin and add the NDVI layer
Map.centerObject(berlin, 10);
Map.addLayer(ndviComposite.clip(berlin), ndviParams, 'NDVI');

// Create the legend
var legend = ui.Panel({
  style: {
    position: 'bottom-left',
    padding: '8px 15px'
  }
});

// Create legend title
var legendTitle = ui.Label({
  value: 'NDVI Legend',
  style: {fontWeight: 'bold', fontSize: '18px', margin: '0 0 4px 0', padding: '0'}
});
legend.add(legendTitle);

// Create the color bar for the legend
var makeColorBar = function(palette) {
  var colorBar = ui.Thumbnail({
    image: ee.Image.pixelLonLat().select(0),
    params: {
      bbox: [0, 0, 1, 0.1],
      dimensions: '100x10',
      format: 'png',
      min: 0,
      max: 1,
      palette: palette,
    },
    style: {stretch: 'horizontal', margin: '0 8px', maxHeight: '24px'},
  });
  return colorBar;
};

// Add the color bar to the legend
legend.add(makeColorBar(ndviParams.palette));

// Create the legend labels
var createLegendLabels = function(values, palette) {
  var labels = ui.Panel({
    widgets: values.map(function(value, index) {
      return ui.Label({
        value: value,
        style: {margin: '4px 8px'}
      });
    }),
    layout: ui.Panel.Layout.flow('horizontal')
  });
  return labels;
};

// Add legend labels
var legendLabels = createLegendLabels(['0', '0.25', '0.5', '1'], ndviParams.palette);
legend.add(legendLabels);

// Add the legend to the map
Map.add(legend);

// Add a histogram of NDVI distribution
var histogram = ui.Chart.image.histogram({
  image: ndviComposite.clip(berlin),
  region: berlin,
  scale: 30,
  minBucketWidth: 0.01
}).setOptions({
  title: 'NDVI Histogram',
  hAxis: {title: 'NDVI'},
  vAxis: {title: 'Pixel count'},
  colors: ['#6A0DAD']
});
print(histogram);

// Define NDVI threshold for urban green areas
var ndviThreshold = 0.3;

// Create a mask for urban green areas
var urbanGreen = ndviComposite.gt(ndviThreshold).selfMask();

// Add urban green areas layer to the map
Map.addLayer(urbanGreen.clip(berlin), {palette: ['green']}, 'Urban Green Areas');

// Calculate the total area of urban green areas
var pixelArea = ee.Image.pixelArea();
var urbanGreenArea = urbanGreen.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: berlin,
  scale: 10,
  maxPixels: 1e9
});

var urbanGreenAreaHa = ee.Number(urbanGreenArea.get('NDVI')).divide(10000);
print('Total Urban Green Area (ha):', urbanGreenAreaHa);

// Optionally export the NDVI composite to Google Drive
Export.image.toDrive({
  image: ndviComposite.clip(berlin),
  description: 'Berlin_NDVI',
  scale: 10,
  region: berlin,
  maxPixels: 1e9
});
