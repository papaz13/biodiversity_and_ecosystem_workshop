---
layout: page
title: A4_MapBiomas
parent: "01-GoogleEarthEngine"
nav_order: 5
---

## A4 Cómo integrar datos de MapBiomas en Google Earth usando Google Earth Engine


## Objetivos

1. Identificar las áreas protegidas vecinas y su distancia al área elegida.
2. Caracterizar el paisaje dentro del área y en su franja: porcentaje de tierra natural, presión humana 2020 y cambio de presión 2000–2020.
3. (Opcional) Calcular el corredor de menor costo hacia un área 

## Datos

- **`WCMC/WDPA/current/polygons`** — Áreas protegidas (WDPA).<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/polygons')`

- **`WRI/SBTN/naturalLands/v1_1/2020`** — Mapa de tierras naturales de SBTN (30 m), banda `natural`: 1 = natural, 0 = no natural.<br>
Image: `ee.Image('WRI/SBTN/naturalLands/v1_1/2020')`

- **`projects/sat-io/open-datasets/GHM/HM_1990_2020_OVERALL_300M`** — Índice de modificación humana global (gHM v3, Theobald et al. 2025), 300 m, de 0 (sin modificación) a 1 (máxima). Catálogo comunitario.<br>
Collection: `ee.ImageCollection('projects/sat-io/open-datasets/GHM/HM_1990_2020_OVERALL_300M')`

- **`FAO/GAUL/2015/level0`** — Límites de países.<br>
Collection: `ee.FeatureCollection('FAO/GAUL/2015/level0')`

> **Nota técnica — dependencia comunitaria:** el gHM proviene del catálogo comunitario (`projects/sat-io/...`), no del catálogo oficial de Earth Engine. El script asume que la colección tiene una propiedad `year` y que la primera banda es el índice. Si el asset cambia de estructura o de ubicación, las capas `hm2000` y `hm2020` fallarán en cascada (paisaje, corredor y escenario B).

# MapBiomas Perú – LULC en Google Earth Engine

Script para visualizar el mapa de Cobertura y Uso del Suelo de MapBiomas Perú
(Colección 1) para un año específico y obtener su URL de teselas XYZ para abrir en Google Earth en web y adicionarlo como proyecto de referencia.

## Uso
1. Abre el [Code Editor de GEE](https://code.earthengine.google.com/).
2. Pega el script y cambia o el nombre del país, y`YEAR` si lo necesitas.
3. Ejecuta: la capa se agrega al mapa y la URL XYZ aparece en la consola.

## Datos
Asset: `projects/mapbiomas-public/assets/peru/lulc/v1`
Fuente: [MapBiomas Perú](https://mapbiomas.org/)

/**
 * MapBiomas Perú – Cobertura y Uso del Suelo (Colección 1)
 * Visualiza un año del mapa LULC en Google Earth Engine
 * e imprime la URL de teselas XYZ.
 */

var ASSET = 'projects/mapbiomas-public/assets/peru/lulc/v1';
var YEAR  = 2024;

var lulc = ee.ImageCollection(ASSET).filter(ee.Filter.eq('year', YEAR));

// Clase: color
var CLASSES = {
  1:  '1f8d49', // Natural forest
  3:  '1f8d49', // Forest
  4:  '7dc975', // Open forest
  5:  '04381d', // Mangrove
  6:  '026975', // Flooded forest
  9:  '7a5900', // Silviculture
  10: 'd6bc74', // Non-forest natural formation
  11: '519799', // Non-forest wetland
  12: 'd6bc74', // Grassland
  13: 'd89f5c', // Other non-forest natural formation
  14: 'ffefc3', // Farming and silviculture
  21: 'ffefc3', // Mosaic of uses
  22: 'd4271e', // Non-vegetated area
  23: 'ffa07a', // Beach, dune and sand spot
  24: 'd4271e', // Urban infrastructure
  25: 'db4d4f', // Other anthropic non-vegetated area
  26: '2532e4', // Water
  27: 'ffffff', // Not observed
  29: 'ffaa5f', // Rocky outcrop
  30: '9c0027', // Mining
  31: '091077', // Aquaculture
  33: '2532e4', // River, lake and ocean
  34: '93dfe6', // Glacier
  68: 'e97a7a', // Other natural non-vegetated area
  74: 'be83f7', // Banana
  81: 'dfeb62', // Andean herbaceous and shrubby vegetation
  82: '6fc179'  // Flooded Andean herbaceous
};

// Construye la paleta 0–82 (clases sin uso = negro)
var palette = [];
for (var i = 0; i <= 82; i++) palette.push(CLASSES[i] || '000000');

var vis = {min: 0, max: 82, palette: palette};

Map.addLayer(lulc, vis, 'MapBiomas Perú LULC ' + YEAR);
Map.centerObject(lulc, 7);

// URL de teselas XYZ
lulc.getMapId(vis, function (mapId) {
  print('Tile URL (XYZ):', mapId.urlFormat);
});

## Script "`LandCover_MapBiomas`":
https://code.earthengine.google.com/00c235ba1bc4bcb5870c3f4e7220e860

