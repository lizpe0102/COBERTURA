# COBERTURA
// Definir un Bounding Box para Bogotá (Aproximadamente)
var bogotaBBox = ee.Geometry.Rectangle([-74.25, 4.45, -73.99, 4.83]);

// Agregar la capa del Bounding Box al mapa
Map.addLayer(bogotaBBox, {color: 'red'}, 'Bounding Box Bogotá');

// Centrar el mapa en Bogotá
Map.centerObject(bogotaBBox, 10);

// Cargar la colección de imágenes Landsat 9, filtrar por fecha y área
var myImage = ee.ImageCollection("LANDSAT/LC09/C02/T1")
  .filterDate('2024-01-01', '2025-01-01')
  .filterBounds(bogotaBBox)
  .sort('CLOUD_COVER')
  .first();

// Verificar si se encontró una imagen
if (myImage) {
  // Recortar la imagen al Bounding Box
  myImage = myImage.clip(bogotaBBox);

  // ---- Cálculo de Índices ----

  // NDVI: (NIR - RED) / (NIR + RED)
  var ndvi = myImage.normalizedDifference(["B5", "B4"]).rename("NDVI").clip(bogotaBBox);

  // Parámetro de ajuste L para SAVI (se usa 0.5 por defecto)
  var L = 0.5;

  // SAVI: ((NIR - RED) / (NIR + RED + L)) * (1 + L)
  var savi = myImage.expression(
    "((NIR - RED) / (NIR + RED + L)) * (1 + L)", {
      "NIR": myImage.select("B5"),
      "RED": myImage.select("B4"),
      "L": L
    }).rename("SAVI").clip(bogotaBBox);

  // NDWI: (GREEN - NIR) / (GREEN + NIR)
  var ndwi = myImage.normalizedDifference(["B3", "B5"]).rename("NDWI").clip(bogotaBBox);

  // ---- Parámetros de Visualización ----

  // Imagen en color natural (RGB)
  var visParams = {
    bands: ["B4", "B3", "B2"],
    gamma: 1.4,
    min: 0,
    max: 30000
  };
  // Paleta de colores para NDVI y SAVI (verde = vegetación, azul = suelo desnudo)
  var vegPalette = {
    min: -1,
    max: 1,
    palette: ['blue', 'white', 'green']
  };

  // Paleta de colores para NDWI (marrón = suelo seco, azul = agua)
  var waterPalette = {
    min: -1,
    max: 1,
    palette: ['brown', 'white', 'blue']
  };

  // ---- Agregar Capas al Mapa ----
  Map.addLayer(myImage, visParams, "Imagen LANDSAT 9");
  Map.addLayer(ndvi, vegPalette, "NDVI");
  Map.addLayer(savi, vegPalette, "SAVI");
  Map.addLayer(ndwi, waterPalette, "NDWI");

  // Centrar el mapa en el Bounding Box de Bogotá
  Map.centerObject(bogotaBBox, 10);
} else {
  print("No se encontró ninguna imagen en el rango de fechas seleccionado.");
}
function addLegend(palette, title, position) {
  var legend = ui.Panel({
    style: {
      position: position,
      padding: '8px 15px'
    }
  });

  var legendTitle = ui.Label({
    value: title,
    style: {fontWeight: 'bold', fontSize: '16px', margin: '0 0 4px 0'}
  });

  legend.add(legendTitle);

  var paletteColors = palette.palette;
  var min = palette.min;
  var max = palette.max;
  var step = (max - min) / (paletteColors.length - 1);

  paletteColors.forEach(function(color, index) {
    var value = min + index * step;
    var colorBox = ui.Label({
      style: {
        backgroundColor: color,
        padding: '8px',
        margin: '0 0 4px 0',
        width: '20px'
      }
    });
    var description = ui.Label({
      value: value.toFixed(2),
      style: {margin: '0 0 4px 4px'}
    });

    legend.add(ui.Panel([colorBox, description], ui.Panel.Layout.Flow('horizontal')));
  });

  Map.add(legend);
}