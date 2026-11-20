---
layout: page
title: A4_MapBiomas
parent: "01-GoogleEarthEngine"
nav_order: 5
---

## MapBiomas_LULC

Este script visualiza el mapa de **cobertura y uso del suelo (LULC) de MapBiomas** para un año elegido, con la paleta oficial de clases, y genera la **URL de teselas XYZ** del mapa para usarla fuera de Earth Engine (Google Earth, QGIS, Leaflet, OpenLayers u otros visores web).

## Objetivos

1. Cargar la colección LULC de MapBiomas (Colección V1) y filtrarla por año.
2. Definir la leyenda de clases (código, nombre y color) en un solo lugar.
3. Construir automáticamente la paleta de visualización a partir de la leyenda.
4. Mostrar el mapa en el Code Editor y centrarlo en el área del país.
5. Generar la URL de teselas XYZ para usar el mapa en otras herramientas.

## Datos

- **`projects/mapbiomas-public/assets/peru/lulc/v1`** — Cobertura y uso del suelo de MapBiomas Perú, Colección V1 (una imagen por año).  
  Collection: `ee.ImageCollection('projects/mapbiomas-public/assets/peru/lulc/v1')`

## Métodos

1. Filtrado de la colección por la propiedad `year` y unión de las imágenes resultantes con `mosaic()`.
2. Definición de la leyenda como diccionario `{código: {nombre, color}}`.
3. Construcción de una paleta continua de 0 al código máximo; los códigos sin clase se rellenan con negro.
4. Visualización con `Map.addLayer` y obtención de la URL XYZ con `getMapId`.

## Paso 1 — Parámetros

```javascript
var RUTA_ASSET = 'projects/mapbiomas-public/assets/peru/lulc/v1';
var ANIO = 2024;
var ZOOM = 7;
var NOMBRE_CAPA = 'MapBiomas LULC';
```

**Parámetros:**

- **`RUTA_ASSET`** — ruta de la colección de MapBiomas. Esta ruta corresponde a **Perú**; para otro país, cambiar `peru` por el país correspondiente y revisar que la leyenda coincida.
- **`ANIO`** — año del mapa a visualizar.
- **`ZOOM`** — nivel de acercamiento inicial del mapa (7 es adecuado para ver un país completo).
- **`NOMBRE_CAPA`** — nombre con el que aparece la capa en el panel de capas; se le agrega el año.

## Paso 2 — Colección del año

```javascript
var coleccion = ee.ImageCollection(RUTA_ASSET)
  .filter(ee.Filter.eq('year', ANIO));

print('Imágenes encontradas para ' + ANIO + ':', coleccion.size());

var lulc = coleccion.mosaic();
```

**Parámetros:**

- **`ee.Filter.eq('year', ANIO)`** — se queda solo con las imágenes cuya propiedad `year` coincide con el año elegido.
- **`coleccion.size()`** — si imprime 0, el año no está disponible en la colección; revisar `ANIO` o `RUTA_ASSET`.
- **`mosaic()`** — une las imágenes del año en una sola. Así `Map.addLayer` y `getMapId` trabajan sobre una imagen y no sobre una colección.

## Paso 3 — Leyenda y paleta

```javascript
var CLASES = {
  1:  {nombre: 'Natural forest',                           color: '1f8d49'},
  3:  {nombre: 'Forest',                                   color: '1f8d49'},
  4:  {nombre: 'Open forest',                              color: '7dc975'},
  5:  {nombre: 'Mangrove',                                 color: '04381d'},
  6:  {nombre: 'Flooded forest',                           color: '026975'},
  9:  {nombre: 'Silviculture',                             color: '7a5900'},
  10: {nombre: 'Non-forest natural formation',             color: 'd6bc74'},
  11: {nombre: 'Non-forest wetland',                       color: '519799'},
  12: {nombre: 'Grassland',                                color: 'd6bc74'},
  13: {nombre: 'Other non-forest natural formation',       color: 'd89f5c'},
  14: {nombre: 'Farming and silviculture',                 color: 'ffefc3'},
  21: {nombre: 'Mosaic of uses',                           color: 'ffefc3'},
  22: {nombre: 'Non-vegetated area',                       color: 'd4271e'},
  23: {nombre: 'Beach, dune and sand spot',                color: 'ffa07a'},
  24: {nombre: 'Urban infrastructure',                     color: 'd4271e'},
  25: {nombre: 'Other anthropic non-vegetated area',       color: 'db4d4f'},
  26: {nombre: 'Water',                                    color: '2532e4'},
  27: {nombre: 'Not observed',                             color: 'ffffff'},
  29: {nombre: 'Rocky outcrop',                            color: 'ffaa5f'},
  30: {nombre: 'Mining',                                   color: '9c0027'},
  31: {nombre: 'Aquaculture',                              color: '091077'},
  33: {nombre: 'River, lake and ocean',                    color: '2532e4'},
  34: {nombre: 'Glacier',                                  color: '93dfe6'},
  68: {nombre: 'Other natural non-vegetated area',         color: 'e97a7a'},
  74: {nombre: 'Banana',                                   color: 'be83f7'},
  81: {nombre: 'Andean herbaceous and shrubby vegetation', color: 'dfeb62'},
  82: {nombre: 'Flooded Andean herbaceous',                color: '6fc179'}
};

var CODIGO_MAX = 82;

function construirPaleta(clases, maximo, colorVacio) {
  var paleta = [];
  for (var i = 0; i <= maximo; i++) {
    paleta.push(clases[i] ? clases[i].color : colorVacio);
  }
  return paleta;
}

var visParams = {
  min: 0,
  max: CODIGO_MAX,
  palette: construirPaleta(CLASES, CODIGO_MAX, '000000')
};
```

**Parámetros:**

- **`CLASES`** — leyenda de MapBiomas: a cada código de píxel le corresponde un nombre y un color. Para agregar o cambiar una clase basta con editar este diccionario.
- **`CODIGO_MAX = 82`** — código más alto de la leyenda; define el largo de la paleta.
- **`construirPaleta`** — arma una lista de colores donde la posición `i` es el color del código `i`. Los códigos que no existen en la leyenda (por ejemplo 2, 7 o 15) quedan en negro (`'000000'`).
- **`min: 0` y `max: CODIGO_MAX`** — junto con una paleta de 83 colores, garantizan que cada código entero reciba exactamente su color, sin interpolación entre clases.

## Paso 4 — Mapa

```javascript
Map.addLayer(lulc, visParams, NOMBRE_CAPA + ' ' + ANIO);
Map.centerObject(coleccion, ZOOM);
```

**Parámetros:**

- **`Map.addLayer`** — agrega el mapa LULC con la paleta del paso 3.
- **`Map.centerObject(coleccion, ZOOM)`** — centra la vista en la extensión de las imágenes del año; se usa la colección porque conserva la huella de las imágenes originales, a diferencia del mosaico.

## Paso 5 — URL de teselas XYZ

```javascript
lulc.getMapId(visParams, function(mapId) {
  print('Tile URL Format (XYZ):', mapId.urlFormat);
});
```

**Parámetros:**

- **`getMapId`** — pide a Earth Engine que genere teselas del mapa con la visualización indicada y devuelve, entre otros datos, la plantilla de URL.
- **`mapId.urlFormat`** — plantilla del tipo `https://earthengine.googleapis.com/.../tiles/{z}/{x}/{y}`. Se puede pegar en QGIS (*XYZ Tiles → Nueva conexión*) o en una capa de teselas de Leaflet u OpenLayers.
- La función se ejecuta de forma **asíncrona** (callback): el resultado aparece en la consola cuando Earth Engine termina de generarlo.


## Script "`MapBiomas_LULC`":

[https://code.earthengine.google.com/efba708f0042ccef51fc4b50349e7fbc](https://code.earthengine.google.com/efba708f0042ccef51fc4b50349e7fbc)
