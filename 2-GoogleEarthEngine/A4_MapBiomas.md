---
layout: page
title: A4_MapBiomas
parent: "01-GoogleEarthEngine"
nav_order: 5
---

## A4 Cómo integrar datos de MapBiomas en Google Earth usando Google Earth Engine

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
 * ============================================================
 *  MapBiomas Perú – Cobertura y Uso del Suelo (Colección 1)
 * ============================================================
 *  Objetivo : Visualizar el mapa LULC de un año y obtener
 *             su URL de teselas XYZ.
 *  Datos    : MapBiomas Perú (https://peru.mapbiomas.org/)
 *  Entorno  : Google Earth Engine Code Editor
 * ============================================================
 */


// ------------------------------------------------------------
// 1. PARÁMETROS
// ------------------------------------------------------------
var ASSET = 'projects/mapbiomas-public/assets/peru/lulc/v1';
var YEAR  = 2024;


// ------------------------------------------------------------
// 2. CARGA DE DATOS
// ------------------------------------------------------------
var lulc = ee.ImageCollection(ASSET)
  .filter(ee.Filter.eq('year', YEAR));


//

## Script "`LandCover_MapBiomas`":
https://code.earthengine.google.com/00c235ba1bc4bcb5870c3f4e7220e860

