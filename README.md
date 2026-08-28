# Metro-Detroit-Lst-Change
This project uses Google Earth Engine to calculate changes in land-surface temperature across Wayne, Oakland, and Macomb counties by using Landsat 8/9.
the analysis compares seasonal land-surface temperature between November 2015-April 2016 and November 2025-April 2026.

Objectives
* Defining the Metro Detroit study area using TIGER: US Census Counties 2018
* Process Landsat thermal imagery using Google Earth Engine
* Remove cloud and cloud-shadow pixels using the QA_PIXEL band.
* Convert Landsat surface temperature data from Kelvin to Celsius.
* Generate median LST composites for each study period.
* Calculate spatial changes in LST between the two periods.
* Export the resulting LST difference raster for future GIS Analysis.

Study Area
the study area consist of
* Wayne County
* Macomb County
* Oakland County
  hese counties encompass the core metropolitan Detroit region.

Data
Landsat 8
* USGS Landsat 8 Level 2, Collection 2, Tier 1
* LANDSAT/LC09/C02/T1_L2
* November 2015-March 2016

Landsat 9
* USGS Landsat 9 Level 2, Collection 2, Tier 1
* LANDSAT/LC09/C02/T1_L2
* November 2025 - March 2026

County Boundaries
* U.S. Census TIGER 2018 Counties

Methods
1. Study Area Selection
The Metro Detorit study area was created by filtering the Michigan county boundaries using the STATEFP and NAME attributes.
2. Cloud Masking
Cloud Masking was done using Landsat QA_PIXEL quality-assurance band
4. Land Surface Temperature
The Landsat ST_B10 surface-temperature band was converted from Kelvin to Celsius using the Collection 2 scaling factors.
6. Seasonal Composites
A median composite was created for each observation period to reduce the influence of individuals observations and produce representative seasonal LST surface.
8. Change Detection
LST change was calculated as:
LST Change = 2025-2026 LST - 2015-2016 LST
Therefore:
* Positive values indicate warmer surface temperature in 2025-2026.
* Negative values indicate cooler surface temperature in 2025-2026.
* Values near zero indicate little change.

Results
The resulting raster provides a spatial representation of land-surface temperature differencs across Metro Detroit.
The analysis produced a mean LST difference across the study area of approximately −6.47°C.
This result represents the difference between the two analyzed seasonal observation periods and should not be interpreted by itself as a long-term climate trend.
Tools
* Google Earth Engine
* JavaScript
* Landsat 8/9

Limitations
This analysis compares two specific seasonal periods rather than a continous long-term anomaly. Differences in weather conditions, snow cover, image availability, adn the timing of satellite observation may influence the observed LST differences.


var metroDetroit = counties.filter(
  ee.Filter.eq('STATEFP', '26')
  ).filter(
    ee.Filter.inList('NAME', ['Wayne', 'Oakland', 'Macomb'])
    );
    print('Metro Detroit');
    
  Map.centerObject(metroDetroit, 9);
  Map.addLayer(metroDetroit, {color: 'black'}, 'Metro Detroit');
                                 Masking Clouds in the image by using QA_PIXEL
function maskClouds(image) {
  var qa = image.select('QA_PIXEL');
  var cloudShadowMask = qa.bitwiseAnd(1 << 4).eq (0);
  var cloudMask = qa.bitwiseAnd(1 << 3).eq(0);
  return image.updateMask(cloudShadowMask). updateMask(cloudMask);
}
                                    Surface Temperature Conversion
function convertToCelsius(image) {
  var LstCelsius = image.select('ST_B10')
  .multiply(0.00341802)
  .add(149.0)
  .subtract(273.15)
  .rename('LST_C');
  return image.addBands(LstCelsius);
}

var LST8 = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2")
.filterBounds(metroDetroit)
.filterDate('2015-11-01', '2016-04-01')
.map(maskClouds)
.map(convertToCelsius);

var LST_2015_2016 = LST8.select('LST_C').median().clip(metroDetroit);

var LST9 = ee.ImageCollection("LANDSAT/LC09/C02/T1_L2")
.filterBounds(metroDetroit)
.filterDate('2025-11-01', '2026-04-01')
.map(maskClouds)
.map(convertToCelsius);

var LST_2025_2026 = LST9.select('LST_C').median().clip(metroDetroit);

var LST_difference = LST_2025_2026.subtract(LST_2015_2016);

var tempVis = {
  min:-10,
  max:30,
  palette: ['blue', 'white', 'red']
};

var diffVis = {
  min:-5,
  max: 5,
  palette: ['blue', 'white', 'red']
};

Map.addLayer(LST_2015_2016, tempVis, 'LST8 (2015-2016)');
Map.addLayer(LST_2025_2026, tempVis, 'LST9 (2025-2026)');
Map.addLayer(LST_difference, diffVis, 'LST Difference (2026 minus 2016)');

Export.image.toDrive({
  image: LST_difference,
  description: 'Detroit_LST_Diff_2016_2026',
  scale: 30, 
  region: metroDetroit.geometry(),
  maxPixels:1e9
});

var stats = LST_difference.reduceRegion({
reducer: ee.Reducer.mean()
.combine({
  reducer2: ee.Reducer.minMax(),
  sharedInputs: true
}),
geometry:metroDetroit.geometry(),
scale: 30,
maxPixels:1e9
});
print('Metro Detroit LST Change Statistics:', stats);
