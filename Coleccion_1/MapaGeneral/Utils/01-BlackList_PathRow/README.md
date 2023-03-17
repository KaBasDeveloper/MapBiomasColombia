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
### Función Factor de escala: Este proceso se debe aplicar a las colecciones 1 y 2 de nivel-2 antes de ser usados .
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

### Función Obtener Colección
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
### Función Porcentaje Nubes
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
### Funcion empleando el metodo TDOM función para la generación de áreas de sombras de nubes.

* @name
 *      tdom
 * @description
 *      El método TDOM calcula primero la media y la desviación estándar de las bandas del infrarrojo cercano (NIR) y del infrarrojo de onda corta (SWIR1) en una colección de imágenes (NIR) y del infrarrojo de onda corta (SWIR1) en una colección de imágenes. Para cada imagen, el algoritmo calcula el valor-z de las bandas NIR y SWIR1 (z(Mb)). Cada imagen también tiene una métrica de oscuridad calculada como la suma de las bandas NIR y SWIR1. A continuación se identifican las sombras de las nubes si un píxel tiene un valor-z inferior a -1 para las bandas NIR y SWIR1 y un valor de oscuridad. Estos umbrales se eligieron tras una amplia evaluación cualitativa de los resultados del TDOM en todo el CONUS.
*  * https://www.mdpi.com/2072-4292/10/8/1184/htm
 * @argument
 *      Objeto que contiene los siguientes atributos
 *          @atributo collection {ee.ImageCollection}
 *          @atributo zScoreThresh {Float}
 *          @atributo shadowSumThresh {Float}
 *          @atributo dilatePixels {Integer}
 * @returns
 *      ee.ImageCollection
 *
```javascript
var tdom = function (obj) {
  
  var shadowSumBands = ['nir', 'swir1'];

  // Obtener media y desviación estándar de píxeles para la serie temporal
  var irStdDev = obj.collection
      .select(shadowSumBands)
      .reduce(ee.Reducer.stdDev());

  var irMean = obj.collection
      .select(shadowSumBands)
      .mean();

  // Enmascarar los valores atípicos oscuros
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
### Funcion CloudProject que permite combinar el metodo anterior (TDOM) y otros procesos para la obtención de mascara de sombras.
* @name
 *      cloudProject
 * @description
 *      Obtencion de mascara de sombras de los diferentes elementos de la imagen.
 * @argument
 *      Objeto que contiene los siguientes atributos
 *          @atributo image {ee.Image}
 *          @atributo cloudHeights {ee.List}
 *          @atributo shadowSumThresh {Float}
 *          @atributo dilatePixels {Integer}
 *          @atributo cloudBand {String}
 * @returns
 *      ee.Image

```javascript
var cloudProject = function (obj) {

  // Obtener la mascara de nubes
  var cloud = obj.image.select(obj.cloudBand);

  // Obtener mascara TDOM
  var tdomMask = obj.image.select(['tdomMask']);
// Proyecta la sombra encontrando píxeles dentro de la máscara TDOM que sean oscuros y   // dentro del área esperada dada la geometría sola, encontrar pixeles oscuros
var darkPixels = obj.image.select(['nir', 'swir1', 'swir2'])
    .reduce(ee.Reducer.sum())
    .lt(obj.shadowSumThresh);

  // Obtener resolucion especial de la imagen
  var nominalScale = cloud.projection().nominalScale();

  // Encuentra donde deben estar las sombras de las nubes en base a la geometría solar
  // Convertir a radianes   
  var meanAzimuth = obj.image.get('sun_azimuth_angle');
  var meanElevation = obj.image.get('sun_elevation_angle');
  var azR = ee.Number(meanAzimuth)
    .multiply(Math.PI)
    .divide(180.0)
    .add(ee.Number(0.5).multiply(Math.PI));
  var zenR = ee.Number(0.5)
    .multiply(Math.PI)
    .subtract(ee.Number(meanElevation).multiply(Math.PI).divide(180.0));

  //Encontrar las sombras
  var shadows = obj.cloudHeights.map(
    function (cloudHeight) {
      cloudHeight = ee.Number(cloudHeight);
      var shadowCastedDistance = zenR.tan()
        .multiply(cloudHeight); //Distance shadow is cast
      var x = azR.cos().multiply(shadowCastedDistance)
        .divide(nominalScale).round(); //X distance of shadow
      var y = azR.sin().multiply(shadowCastedDistance)
        .divide(nominalScale).round(); //Y distance of shadow

      return cloud.changeProj(cloud.projection(), cloud.projection()
        .translate(x, y));
    }
  );

  var shadow = ee.ImageCollection.fromImages(shadows).max().unmask();

  //Crear mascara de sombras
  shadow = shadow.focal_max(obj.dilatePixels);
  shadow = shadow.and(darkPixels).and(tdomMask.not().and(cloud.not()));

  var shadowMask = shadow.rename(['shadowTdomMask']);

  return obj.image.addBands(shadowMask);
};
```

### Funcion cloudBQAMaskSr que permite tomar los pixeles validos para la mascara de nubes.
* @name
 *      cloudBQAMaskSr
 * @description
 *      Tomar pixeles de calidad generados por el algoritmos CFMASK para la imagen.
 * @argument
 *      @image {ee.Image}
 * @returns
 *      ee.Image

```javascript
var cloudBQAMaskSr = function (image) {
  //Seleccion pixeles de calidad
  var qaBand = image.select(['pixel_qa']);
  //Filtrado pixeles
  var cloudMask = qaBand.bitwiseAnd(Math.pow(2, 3))
    .or(qaBand.bitwiseAnd(Math.pow(2, 2)))
    .or(qaBand.bitwiseAnd(Math.pow(2, 1)))
    .neq(0)
    .rename('cloudBQAMask');
  return ee.Image(cloudMask);
};
var cloudBQAMask = function (image) {
  var cloudMask = cloudBQAMaskSr(image);
  return image.addBands(ee.Image(cloudMask));
};
```

### Funcion shadowBQAMaskSrLX que permite tomar los pixeles validos para la mascara de sombras.
* @name
 *      shadowBQASrLX
 * @description
 *      Tomar pixeles de calidad para la mascara de sombras.
 * @argument
 *      @image {ee.Image}
 * @returns
 *      ee.Image

```javascript
var shadowBQAMaskSrLX = function (image) {
  var qaBand = image.select(['pixel_qa']);
  var cloudShadowMask = qaBand
    .bitwiseAnd(Math.pow(2, 4))
    .neq(0)
    .rename('shadowBQAMask');
  return ee.Image(cloudShadowMask);  
};
var shadowBQAMask = function (image) {
    var cloudShadowMask = ee.Algorithms.If(
        ee.String(image.get('satellite_name')).slice(0, 10).compareTo('Sentinel-2').not(),
        // true
        ee.Image(0).mask(image.select(0)).rename('shadowBQAMask'),
        // false
        shadowBQAMaskSrLX(image)
    );
    return image.addBands(ee.Image(cloudShadowMask));
};
```

### Funcion getMasks que permite aplicar la mascara de nubes y sombras para la imagen de interés.
* @name
 *      getMasks
 * @description
 *      Obtener y aplicar mascara de nubes y sombras sobre la coleccion
 * @argument
 *      Objecto contendo os atributos
 *          @attribute collection {ee.ImageCollection}
 *          @attribute cloudBQA {Boolean}
 *          @attribute cloudScore {Boolean}
 *          @attribute shadowBQA {Boolean}
 *          @attribute shadowTdom {Boolean}
 *          @attribute zScoreThresh { Float}
 *          @attribute shadowSumThresh { Float}
 *          @attribute dilatePixels { Integer}
 *          @attribute cloudHeights {ee.List}
 *          @attribute cloudBand {String}
 * @returns
 *      ee.ImageCollection
```javascript
var getMasks = function (obj) {
  //Obtener mascara de sombras 
  var getShadowTdomMask = function (image) {
    image = cloudProject({
      'image': image,
      'shadowSumThresh': obj.shadowSumThresh,
      'dilatePixels': obj.dilatePixels,
      'cloudHeights': obj.cloudHeights,
      'cloudBand': obj.cloudBand,
    });
    return image;
  };
  // Obtener mascara de nubes
  var collection = ee.ImageCollection(
    ee.Algorithms.If(
      obj.cloudBQA, ee.Algorithms.If(
        obj.cloudScore,
        obj.collection.map(cloudBQAMask).map(cloudScore),
        obj.collection.map(cloudBQAMask)
      ),
      obj.collection.map(cloudScore)
    )
  );
  // Combinacion mascara de nubes y sombras
  collection = ee.ImageCollection(
    ee.Algorithms.If(
      obj.shadowBQA,
      ee.Algorithms.If(
        obj.shadowTdom,
        tdom({
          'collection': collection.map(shadowBQAMask),
          'zScoreThresh': obj.zScoreThresh,
          'shadowSumThresh': obj.shadowSumThresh,
          'dilatePixels': obj.dilatePixels,
        }),
        collection.map(shadowBQAMask)
      ),
      tdom({
        'collection': collection,
        'zScoreThresh': obj.zScoreThresh,
        'shadowSumThresh': obj.shadowSumThresh,
        'dilatePixels': obj.dilatePixels,
      })
    )
  );


  return ee.ImageCollection(
    ee.Algorithms.If(
      obj.shadowTdom,
      collection.map(getShadowTdomMask),
      collection)
  );
};
```

### Funcion getImages que permite obtener las imágenes con base a los parámetros establecidos y aplicando las funciones anteriormente indicadas para el enmascaramiento de sombras y nubes, también se excluyen las imágenes indicadas en la lista “blacklist”.
* @name
 *      getImages
 * @description
 *      Obtener el listado de imágenes según los parámetros definidos
 * @argument
 *      Objeto que contiene los siguientes atributos
 *          @attribute grid_name {String}
 *          @attribute t0 {Date}
 *          @attribute t1 {Date}
 *          @attribute satellite {String}
 *          @attribute cloud_cover {Int}
 *          @attribute pais {String}
 *          @attribute regionMosaic {Int}
 *          @attribute shadowSum {Int}
 *          @attribute cloudThresh {Int}
 *      @attribute blacklist {ee.List}
 * @returns
 *      ee.ImageCollection

```javascript
var getImages = function (param, blackList) {
  //Cargue de variables y definicion de listas segun el sensor
  var options = {
    dates: {
      t0: param.t0,
      t1: param.t1
    },
    collection: null,
    regionMosaic: param.regionMosaic,
    gridName: param.grid_name,
    cloudCover: param.cloud_cover,
    shadowSum: param.shadowSum,
    cloudThresh: param.cloudThresh,
    blackList: blackList,
    imageList: [],
    collectionid: param.satellite,
    collectionIds: {
      'l4': ['LANDSAT/LT04/C02/T1_L2'],
      'l5': ['LANDSAT/LT05/C02/T1_L2'],
      'l7': ['LANDSAT/LE07/C02/T1_L2'],
      'l8': ['LANDSAT/LC08/C02/T1_L2'],
      'lx': ['LANDSAT/LT05/C02/T1_L2',
        'LANDSAT/LE07/C02/T1_L2'],
      'l9': ['LANDSAT/LC09/C02/T1_L2'],
      'lxx': ['LANDSAT/LC08/C02/T1_L2',
        'LANDSAT/LC09/C02/T1_L2'],
    },
    endmembers: {
      'l4': sma.endmembers['landsat-4'],
      'l5': sma.endmembers['landsat-5'],
      'l7': sma.endmembers['landsat-7'],
      'l8': sma.endmembers['landsat-8'],
      'lx': sma.endmembers['landsat-5'],
      'l9': sma.endmembers['landsat-9'],
      'lxx': sma.endmembers['landsat-8'],
    },
    bqaValue: {
      'l4': ['QA_PIXEL', Math.pow(2, 5)],
      'l5': ['QA_PIXEL', Math.pow(2, 5)],
      'l7': ['QA_PIXEL', Math.pow(2, 5)],
      'l8': ['QA_PIXEL', Math.pow(2, 5)],
      'lx': ['QA_PIXEL', Math.pow(2, 5)],
      'l9': ['QA_PIXEL', Math.pow(2, 5)],
      'lxx': ['QA_PIXEL', Math.pow(2, 5)],
    },
    bandIds: {
      'LANDSAT/LT04/C02/T1_L2': 'l4_sr2',
      'LANDSAT/LT05/C02/T1_L2': 'l5_sr2',
      'LANDSAT/LE07/C02/T1_L2': 'l7_sr2',
      'LANDSAT/LC08/C02/T1_L2': 'l8_sr2',
      'LANDSAT/LC09/C02/T1_L2': 'l9_sr2',
    },
    visParams: {
      bands: 'swir1,nir,red',
      gain: '0.008,0.006,0.02',
      gamma: 0.75
    }
  };
  // Aplicar mascara de nubes y sombras para la coleccion
  var applyCloudAndSahdowMask = function (collection) {
    var collectionWithMasks = getMasks({
      'collection': collection,
      'cloudBQA': true,    // cloud mask using pixel QA
      'cloudScore': true,  // cloud mas using simple cloud score
      'shadowBQA': true,   // cloud shadow mask using pixel QA
      'shadowTdom': true,  // cloud shadow using tdom
      'zScoreThresh': -1,
      'shadowSumThresh': options.shadowSum,
      'cloudThresh': options.cloudThresh,
      'dilatePixels': 2,
      'cloudHeights': ee.List.sequence(2000, 10000, 500),
      'cloudBand': 'cloudScoreMask' //'cloudScoreMask' or 'cloudBQAMask'
    });
    // Obtener la coleccion sin nubes
    var collectionWithoutClouds = collectionWithMasks.map(
      function (image) {
        return image.mask(
          image.select([
            'cloudBQAMask',
            'cloudScoreMask',
            'shadowBQAMask',
            'shadowTdomMask'
          ]).reduce(ee.Reducer.anyNonZero()).eq(0)
        );
      }
    );
    return collectionWithoutClouds;
  };
  //Aplicar mascara de nubes para cada imagen
  var applySingleCloudMask = function (image) {
    return image.mask(
      image
        .select(options.bqaValue[options.collectionid][0])
        .bitwiseAnd(options.bqaValue[options.collectionid][1])
        .not()
    );
  };
  //Obtener coleccion segun los parametros de fecha, nbosidad y region
  var processCollection =  function (collectionid) {
    var spectralBands = ['blue', 'red', 'green', 'nir', 'swir1', 'swir2'];
    var objLandsat = {
      'collectionid': collectionid,
      'gridName':     options.gridName,
      'dateStart':    options.dates.t0.slice(0, 4) + '-01-01',
      'dateEnd':      options.dates.t1.slice(0, 4) + '-12-31',
      'cloudCover':   options.cloudCover,
    };
    var bands = bns.get(options.bandIds[collectionid]);
    var collection = getCollection(objLandsat)
        .select(bands.bandNames, bands.newNames)
        .filter(ee.Filter.inList('system:index', options.blackList).not());
   collection = collection.map(scaleFactors);
   collection = applyCloudAndSahdowMask(collection).select(spectralBands);
   return collection;    
  };
  //Crear coleccion segun los datos obtenidos
  var makeCollection = function () {
    var collection = processCollection(
       options.collectionIds[options.collectionid][0]
    );
    // Unmask data with the secondary mosaic (+L5 or +L7)
    if (options.collectionIds[options.collectionid].length == 2) {
      var collection2 = processCollection(
        options.collectionIds[options.collectionid][1]
      );
      collection = collection.merge(collection2);
   }
    return collection;
  };
  return makeCollection().filterDate(options.dates.t0, options.dates.t1);
};
```

## 2 Cargar modulos
Librería de funciones ya definidas en otros ambientes y empleadas en el proceso
```javascript
var bns = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/BandNames.js');
var dtp = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/DataType.js');
var ind = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/SpectralIndexes.js');
var mis = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/Miscellaneous.js');
var mos = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/Mosaic.js');
var sma = require('users/raisgmb01/MapBiomas_C4:P01_MOSAICOS/modules/SmaAndNdfi.js');
```

## 3 Implementacion
A partir de la función getImages se obtienen dos coleciones una con la lista de imágenes excluidas y otra sin imágenes excluidas
```javascript
var collection_without_blacklist = getImages(param, []);
var collection_with_blacklist = getImages(param, blackList);
print('collection sin blackList:', collection_without_blacklist);
print('collection con blackList:', collection_with_blacklist);
```

## 3.1 Visualizacion
Visualizacion y despliegue de la región de interés y las colecciones resultantes.
```javascript
Map.setOptions({ mapTypeId: 'SATELLITE'});
Map.addLayer(
  ee.FeatureCollection(
    collection_with_blacklist
      .first()
      .geometry()
      //.bounds()
  )
  .style({ fillColor: 'f8fc0300', color: 'f8fc03'}), 
  {}, 
  'GEOMETRY'
);
Map.addLayer(
  collection_without_blacklist.median().clip(region),
  {
    bands: 'swir1,nir,red',
    gain: '0.08,0.06,0.2'
  },
  'MOSAIC',
  true
);                 
Map.addLayer(
  collection_with_blacklist.median().clip(region),
  {
    bands: 'swir1,nir,red',
    gain: '0.08,0.06,0.2'
  },
  'MOSAIC BLACK LIST',
  true 
);
Map.addLayer(
    region.style({ fillColor: '#ff000000', color: 'f59e42'}),
    {}, 'REGIONS', false
);
```
### Despliega las escenas landsat disponibles para cada mosaico, de manera que se puede visualizar cómo afecta la calidad del mosaico
```javascript
collection_with_blacklist.reduceColumns(ee.Reducer.toList(), ['system:index'])
  .get('list')
  .evaluate(
    function(ids){
      ids.forEach(
        function(imageid){
          var image = collection_with_blacklist
            .filterMetadata('system:index', 'equals', imageid)
            .mosaic();
          Map.addLayer(image,
            {
              bands: 'swir1,nir,red',
              gain: '0.08,0.06,0.2'
            },
            imageid,
            false
          );
          print(imageid);
        }
      );
    }  
  );
  

### Despliegue de datos de Global Forest Change  para Colombia.
```javascript
  //=====HANSEN
var ruta = ee.FeatureCollection("projects/mapbiomas-raisg/DATOS_AUXILIARES/VECTORES/RAISG_limites_pais_2018");
var filter = ee.Filter.inList('paisiso', ['COL']);
var Boundary = ruta.filter(filter);

//HANSEN
var dataset = ee.Image("UMD/hansen/global_forest_change_2021_v1_9");
var treeCoverVisParam = {
  bands: ['treecover2000'],
  min: 0,
  max: 100,
  palette: ['black', 'green']
};
//Map.addLayer(dataset, treeCoverVisParam, 'tree cover');

var treeLossVisParam = {
  bands: ['lossyear'],
  min: 18,
  max: 19,
  palette: ['#fff3bf', 'red']
};
//Map.addLayer(dataset, treeLossVisParam, 'HANSEN tree loss 2019');

//- for each - evaluate
Map.addLayer(dataset.select('lossyear').eq(13).selfMask().clip(region), {min:0,max:1,palette: ['#ff4d4d']}, 'HANSEN tree loss 2013',false);
```

## License

[Creative Commons licensed](./LICENSE-docs).
