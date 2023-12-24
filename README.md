// Map zooms to the point of interest.
Map.centerObject(geometry3,15);

// Getting palettes for later and change basemap from Google basemaps.
var palettes = require('users/gena/packages:palettes');
var JMpalette = palettes.colorbrewer.GnBu[7];
var baseChange = [{featureType: 'all', stylers: [{visibility: 'off'}]}];

//Set map option in the display.
Map.setOptions('baseChange', {'baseChange': baseChange});

// Import Jamaica JAD69 shapefile.
var jam = ee.FeatureCollection(jamshp).geometry();
// Add to display.
Map.addLayer(jam, {color:'#666699'}, "JAM")


///////////////////// Requesting Data //////////////////////////


//Create cloud mask function.

function maskS2clouds(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask).divide(10000);
}

// Request Sentinel-2 data with specific parameters.
var start = ee.Date('2021-01-01');
var finish = ee.Date('2021-01-20');
var collection = ee.ImageCollection('COPERNICUS/S2')
                  .filterDate(start,finish)
                  .filterBounds(geometry) // Uses boundary of shapefile.
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10)) //Filters out images with cloud cover >10%.
                  .map(maskS2clouds) // Add cloud mask for existing clouds.
                  .limit(1); // Select only the first image in the collection.

var rgbVis = {bands: ['B8','B4', 'B3']}; // Select image bands from Sentinel-2


// Add function to select and display image in the console and display.

function addImage(imageL) { // Display each image in collection.
  var id = imageL.id;
  var image = ee.Image(imageL.id);
  Map.addLayer(collection.mean(),rgbVis,id)
}

collection.evaluate(function(collection) {  // Use map on client-side.
  print(collection.features);
  collection.features.map(addImage);
})


// Calculate image statistics.

var getStats = function(image) {
  
  var reducers = ee.Reducer.mean().combine({
  reducer2: ee.Reducer.stdDev(),
  sharedInputs: true
  });

  var stats1 = ee.Image(image).reduceRegion({
    reducer: reducers,
    geometry: geometry2,
    scale: 10,
    bestEffort: true
  });
  
  return ee.Image(image).set(stats1);

};


var statistics = collection.map(getStats);

print("statistics", statistics);


// Get SRTM data

var dataset = ee.Image('USGS/SRTMGL1_003');
var elevation = dataset.select('elevation');
var slope = ee.Terrain.slope(elevation).clip(geometry2);


Map.addLayer(slope, {min: 0, max: 60}, 'slope');

//Create a chart.

var chart = ui.Chart.image.byRegion({
  image: elevation,
  regions: geometry3,
  scale: 10
});

// Display the chart in the console.
print(chart);

// Get MERIT data to compare with SRTM 2000 data.

var dataset1 = ee.Image('MERIT/DEM/v1_0_3').clip(geometry2);

var visualization = {
  bands: ['dem']
};

// Map.addLayer(dataset1, visualization, 'Elevation');

//Create a chart.

var chart0 = ui.Chart.image.byRegion({
  image: dataset,
  regions: geometry2,
  scale: 10
});

// Display the chart in the console.
print(chart0);


// Elevation statistics from SRTM data

var img  = ee.Image('USGS/SRTMGL1_003').select('elevation');
var meanElev = img.sample({region: geometry, scale: 25000, numPixels: 100, geometries: true}); //250
print ('Mean elev info:', meanElev)


// Combine the mean and standard deviation reducers.
var reducers = ee.Reducer.mean().combine({
  reducer2: ee.Reducer.stdDev(),
  sharedInputs: true
});

// Use the combined reducer to get the mean and SD of the image.
var stats = img.reduceRegion({
  reducer: reducers,
  geometry: geometry2,
  bestEffort: true,
});

// Display the dictionary of band means and SDs.
print(stats);



/////////////////////// Windalco pit dimensions///////////////////////////////////



//Get area of Windalco red mud pit ROI (region of interest).

var windalcoarea = geometry2.area({'maxError': 1});
print ('Pit area:', windalcoarea)



// This function computes the feature's geometry area and adds it as a property.

var segLength = function(feature) {
  return feature.set({segLength: feature.geometry().length()});
};

// Map the area getting function over the FeatureCollection.
var lengthAdded = geometry4.map(segLength);

// Print the first feature from the collection with the added property.

print('transect lengths:', lengthAdded);
var transectLengths = lengthAdded.aggregate_array("segLength")
print(transectLengths)


//Get statistics

var sum = transectLengths.reduce(ee.Reducer.sum())
print ('Total transect length:',sum)

var mean = transectLengths.reduce(ee.Reducer.mean())
print ('Mean transect length:',mean)

var stdDev = transectLengths.reduce(ee.Reducer.stdDev())
print ('StdDev transect length:',stdDev)

////////////// Export any image to the Drive ////////////

 Export.image.toDrive({
   image: img,
   description: 'pit_elevation',
   scale: 10,
   region: geometry2,
   crs: 'EPSG:24200', //EPSG:3448 for JAD2001
   maxPixels: 1e10
 });



