# 1. Selección de Set de imágenes
Se hace uso del script denominado 01_Black_List_Path_Row en el cual se realiza la selección de las mejores tomas satelitales para la construcción del mosaico de acuerdo al Path y Row de la familia de Satélites de Landsat. 

```javascript
```
## 1.1 Variables de entrada
Se define un arreglo de parámetros que contiene todas las variables de entrada para el script
```javascript
var param = {
    'grid_name': '008057',// Esta variable corresponde al path y row, 009055 = '9-55'
    't0': '2022-04-01', //Inicio de rango de la selección
    't1': '2023-02-28',//Fin del rango de la selección
    'satellite': 'lxx', //Colección o combinación de colecciones (L_5-9, lx. lxx)
    'cloud_cover': 70, // Parámetro que establece el enmascaramiento de nubosidad
    'pais': 'Colombia',  // País
    'regionMosaic': 304  // Región alta o baja
    'shadowSum': 3500,    // 0 - 10000  Defaut 3500  
    'cloudThresh': 10,    // 0 - 100    Defaut 10
};
```
Se define las imágenes que serán excluidas del procesamiento y la región a filtrar según los parámetros iniciales
```javascript
var blackList = [
  // , 'LC08_004057_20220706','LC08_004057_20220807','LC08_004057_20220924',
];
var layers = {
  regions: 'users/kahuertas/Regiones_Mosaico_COL_NoRAISG'
};
var region = ee.FeatureCollection(layers.regions)
  .filterMetadata('RegMosaico', 'equals', param.regionMosaic);
```
## 1.2 Definición de Funciones
Se declaran las funciones a emplear a lo largo del proceso
### Función reescale
* @nombre
*      reescale
* @descripcion
*      Emplear un reescalamiento a la imagen, el cual consiste en cambiar el rango espectral de la imagen
* @argument
*      Objeto que contiene atributos
*          @atributo image {ee.Image}
*          @atributo min {Integer}
*          @atributo max {Integer}
* @returns
*      ee.Image

```javascript
var rescale = function (obj) {
  var image = obj
    .image
    .subtract(obj.min)
    .divide(ee.Number(obj.max)
    .subtract(obj.min));
  return image;
};
```
### Funcion Factor de escala: Este proceso se debe aplicar a las colecciones 1 y 2 de nivel-2 antes de ser usados .
* @nombre
*      scaleFactors
* @descripcion
*      Factor de escala a aplicar a los productos de reflectancia de superficie antes de ser usados.
* @argument
*      @atributo image {ee.Image}
* @returns
*      ee.Image

```javascript
var scaleFactors = function(image) {
  var optical = [
    'blue',  'green', 'red', 'nir', 'swir1', 'swir2'
  ];
  var opticalBands = image
    .select(optical).multiply(0.0000275).add(-0.2).multiply(10000);
  var thermalBand = image
    .select('temp*').multiply(0.00341802).add(149.0);
  
  return image
    .addBands(opticalBands, null, true)
    .addBands(thermalBand, null, true);
};
```

### Funcion Obtener coleccion
* @nombre
*      getCollection
* @descripcion
*      Proceso para la generación y obtención de la colección de imágenes con base a los parámetros establecidos.
* @argument
*      Objeto que contiene atributos
*          @atributo collectionid {String}
*          @atributo gridName {String}
*          @atributo dateStart {Date}
*          @atributo dateEnd {Date}
*          @atributo cloudCover {Number}


* @returns
*      ee.ImageCollection

```javascript
var getCollection = function (obj) { 
  var setProperties = function (image) {
      //Obtencion de datos referents a la cobertura de nubes
      var cloudCover = ee.Algorithms.If(image.get('SPACECRAFT_NAME'),
          image.get('CLOUDY_PIXEL_PERCENTAGE'),
          image.get('CLOUD_COVER')
      );
      //Obtencion de datos referents a la fecha de adquisición de las imagenes
      var date = ee.Algorithms.If(image.get('DATE_ACQUIRED'),
          image.get('DATE_ACQUIRED'),
          ee.Algorithms.If(image.get('SENSING_TIME'),
              image.get('SENSING_TIME'),
              image.get('GENERATION_TIME')
          )
      );
      //Obtencion de datos referents al satélite que tomo la imagen
      var satellite = ee.Algorithms.If(image.get('SPACECRAFT_ID'),
          image.get('SPACECRAFT_ID'),
          ee.Algorithms.If(image.get('SATELLITE'),
              image.get('SATELLITE'),
              image.get('SPACECRAFT_NAME')
          )
      );
      //Obtencion de datos referents a los angulos de azimut del sol en la toma de la imagen
      var azimuth = ee.Algorithms.If(image.get('SUN_AZIMUTH'),
          image.get('SUN_AZIMUTH'),
          ee.Algorithms.If(image.get('SOLAR_AZIMUTH_ANGLE'),
              image.get('SOLAR_AZIMUTH_ANGLE'),
              image.get('MEAN_SOLAR_AZIMUTH_ANGLE')
          )
      );
      //Obtencion de datos referents a la elevación del sol en la toma de la imagen
      var elevation = ee.Algorithms.If(image.get('SUN_ELEVATION'),
          image.get('SUN_ELEVATION'),
          ee.Algorithms.If(image.get('SOLAR_ZENITH_ANGLE'),
              ee.Number(90).subtract(image.get('SOLAR_ZENITH_ANGLE')),
              ee.Number(90).subtract(image.get('MEAN_SOLAR_ZENITH_ANGLE'))
          )
      );
  
      //Obtencion de datos referents a si la imagen es reflectancia de superficie o en el tope de la atmosfera.
      var reflectance = ee.Algorithms.If(
          ee.String(ee.Dictionary(ee.Algorithms.Describe(image)).get('id')).match('SR').length(),
          'SR',
          'TOA'
      );
      //Enviar datos de condiciones de toma y demás a cada una de las imágenes.
      return image
          .set('image_id', image.id())
          .set('cloud_cover', cloudCover)
          .set('satellite_name', satellite)
          .set('sun_azimuth_angle', azimuth)
          .set('sun_elevation_angle', elevation)
          .set('reflectance', reflectance)
          .set('date', ee.Date(date).format('Y-MM-dd'));
  };
  //Definicion de los filtros según los parámetros iniciales
  var filters = ee.Filter.and(
      ee.Filter.date(obj.dateStart, obj.dateEnd),
      ee.Filter.lte('cloud_cover', obj.cloudCover),
      ee.Filter.eq('WRS_PATH', parseInt(obj.gridName.slice(0, 3), 10)),
      ee.Filter.eq('WRS_ROW', parseInt(obj.gridName.slice(3, 6), 10))
  );
  //Obtencion de la colección, enviar las propiedades y filtrar según parámetros.
  var collection = ee.ImageCollection(obj.collectionid)
      .map(setProperties)
      .filter(filters);
  return collection;
};
```
### Puntaje Nubes
* @nombre
*      cloudScore
* @descripcion
*      Proceso para la generación y obtención de la colección de imágenes con base a los parámetros establecidos.
* @argument
*      Objeto que contiene atributos
*          @atributo image {ee.Image}
* @returns
*      ee.Image
```javascript
var cloudScore = function (image) {
  var cloudThresh = param.cloudThresh;
  // Calcular multiples indicadores de  nubosidad y toma el minimo de ellos.
  var score = ee.Image(1.0);
  // Nubes son razonablemente brillantes en la banda del azul
  score = score.min(
    rescale(
      {
        'image': image.select(['blue']),
        'min': 1000,
        'max': 3000
      }
    )
  );
  // Nubes son razonablemente brillantes en todas las bandas del visible.
  score = score.min(
    rescale(
      {
        'image': image.expression("b('red') + b('green') + b('blue')"),
        'min': 2000,
        'max': 8000
      }
    )
  );
  // Nubes son razonablemente brillantes en todas las bandas del infrarrojo.
  score = score.min(
    rescale(
      {
        'image': image.expression("b('nir') + b('swir1') + b('swir2')"),
        'min': 3000,
        'max': 8000
      }
    )
  );
  // Nubes son firas con banda temperatura.
  var temperature = image.select(['temp']);
  score = score.where(temperature.mask(),
      score.min(rescale({
          'image': temperature,
          'min': 300,
          'max': 290
      })));

  // Proceso para separa nubes de nieve.
  var ndsi = image.normalizedDifference(['green', 'swir1']);

  score = score.min(
    rescale(
      {
      'image': ndsi,
      'min': 0.8,
      'max': 0.6
      }
    )
  )
  .multiply(100)
  .byte();
  score = score.gte(cloudThresh).rename('cloudScoreMask');
  return image.addBands(score);
};
```
### Funcion empleando el metido TOM función par la conversión y validación de datos obtenidos.

* @name
 *      tdom
 * @description
 *      El método TDOM calcula primero la media y la desviación estándar de las bandas del infrarrojo cercano (NIR) y del infrarrojo de onda corta (SWIR1) en una colección de imágenes (NIR) y del infrarrojo de onda corta (SWIR1) en una colección de imágenes.Para cada imagen, el algoritmo calcula la puntuación z de las bandas NIR y SWIR1 (z(Mb)). Cada imagen también tiene una métrica de oscuridad calculada como la suma de las bandas NIR y SWIR1. A continuación se identifican las sombras de las nubes si un píxel tiene una puntuación z inferior a -1 para las bandas NIR y SWIR1 y un valor de oscuridad. Estos umbrales se eligieron tras una exhaustiva evaluación cualitativa de los resultados de TDOM de todo el mundo. Estos umbrales se eligieron tras una amplia evaluación cualitativa de los resultados del TDOM en todo el CONUS.
*  * https://www.mdpi.com/2072-4292/10/8/1184/htm
 * @argument
 *      Objecto contendo os atributos
 *          @attribute collection {ee.ImageCollection}
 *          @attribute zScoreThresh {Float}
 *          @attribute shadowSumThresh {Float}
 *          @attribute dilatePixels {Integer}
 * @returns
 *      ee.ImageCollection
 *
```javascript
var tdom = function (obj) {
  
  var shadowSumBands = ['nir', 'swir1'];

  // Get some pixel-wise stats for the time series
  var irStdDev = obj.collection
      .select(shadowSumBands)
      .reduce(ee.Reducer.stdDev());

  var irMean = obj.collection
      .select(shadowSumBands)
      .mean();

  // Mask out dark dark outliers
  var collection = obj.collection.map(
    function (image) {
      var zScore = image.select(shadowSumBands)
        .subtract(irMean)
        .divide(irStdDev);

      var irSum = image.select(shadowSumBands)
        .reduce(ee.Reducer.sum());

      var tdomMask = zScore.lt(obj.zScoreThresh)
        .reduce(ee.Reducer.sum())
        .eq(2)
        .and(irSum.lt(obj.shadowSumThresh))
        .not();

      tdomMask = tdomMask.focal_min(obj.dilatePixels);

      return image.addBands(tdomMask.rename('tdomMask'));
    }
  );

  return collection;
  
};

```


## License

[Creative Commons licensed](./LICENSE-docs).
