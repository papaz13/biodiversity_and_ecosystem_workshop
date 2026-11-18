---
layout: page
title: A1_Representatividad_Ecorregiones
parent: "01-GoogleEarthEngine"
nav_order: 3
---

## A1_Representatividad_Ecorregiones

Este script mide la **representatividad ecológica de las áreas protegidas** en dos escenarios. En el **escenario A** parte de un área elegida y responde, para cada ecorregión que la contiene: qué extensión tiene en el país, cuánto está protegido y cuánto cae dentro del área. En el **escenario B** responde a nivel país: qué porcentaje del territorio está protegido y qué porcentaje de **cada** ecorregión lo está.

## Objetivos

1. Definir el área de interés en uno de tres modos (WDPA, asset propio o dibujo) y detectar el país automáticamente.
2. Construir la capa de áreas protegidas del país con los criterios de Protected Planet.
3. Calcular, por ecorregión, la superficie en el país, la superficie protegida y la superficie dentro del área elegida (escenario A).
4. Calcular el porcentaje del país protegido y el porcentaje protegido de cada ecorregión (escenario B).
5. Comparar cada ecorregión con la referencia de 30 % de la Meta 3.
6. Exportar las tablas a Google Drive cuando el cálculo en consola se demore.

**Aporta a:** Meta 3 del Marco Mundial de Biodiversidad Kunming-Montreal ("ecológicamente representativas"), Indicador principal 3.1 (cobertura de áreas protegidas y OMEC).

## Datos

- **`WCMC/WDPA/current/polygons`** — Áreas protegidas (WDPA).<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/polygons')`

- **`RESOLVE/ECOREGIONS/2017`** — Ecorregiones terrestres del mundo.<br>
Collection: `ee.FeatureCollection('RESOLVE/ECOREGIONS/2017')`

- **`FAO/GAUL/2015/level0`** — Límites de países.<br>
Collection: `ee.FeatureCollection('FAO/GAUL/2015/level0')`

## Métodos

1. Selección del área en tres modos y detección del país con `filterBounds` sobre el centroide del área.
2. Filtrado de áreas protegidas con criterios de Protected Planet (sin propuestas, sin estado no reportado, sin Reservas de Biosfera UNESCO-MAB).
3. Máscaras ráster binarias (país, áreas protegidas, mi área) multiplicadas por el área del píxel; las superposiciones entre áreas se cuentan una sola vez.
4. Rasterización de las ecorregiones con `reduceToImage` sobre `ECO_ID`.
5. Suma agrupada por ecorregión en una sola llamada con `ee.Reducer.sum().repeat(3).group()`.
6. Cálculo de porcentajes y presentación en consola, gráficos y tablas exportables.

## Paso 1 — Parámetros

```javascript
var MODO = 'wdpa';
var SITE_ID = 3370;
var RUTA_ASSET = 'projects/SU-PROYECTO/assets/mi_area';
var PAIS_GAUL = null;
var CALCULAR_PAIS = true;

var ESCALA = 300;
```

**Parámetros:**
- **`MODO`** — `'wdpa'` (usa `SITE_ID`), `'asset'` (usa `RUTA_ASSET`) o `'dibujo'` (usa la geometría `miArea`).
- **`SITE_ID = 3370`** — demostración con la Reserva Nacional Tambopata; el propio se obtiene con el script A0.
- **`PAIS_GAUL`** — `null` detecta el país automáticamente; para forzarlo, escribir el nombre GAUL (por ejemplo `'Peru'`).
- **`CALCULAR_PAIS`** — activa el escenario B (todo el país). Es más pesado; si se demora, usar la exportación del paso 9.
- **`ESCALA = 300`** — resolución de cálculo en metros; suficiente para indicadores a escala de ecorregión y país.

## Paso 2 — Mi área

```javascript
var wdpa = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var miAreaFC;
if (MODO === 'wdpa') {
  miAreaFC = wdpa.filter(ee.Filter.eq('SITE_ID', SITE_ID));
} else if (MODO === 'asset') {
  miAreaFC = ee.FeatureCollection(RUTA_ASSET);
} else if (MODO === 'dibujo') {
  if (typeof miArea === 'undefined') {
    throw new Error('MODO = "dibujo": dibuje un polígono en el mapa y nómbrelo miArea.');
  }
  miAreaFC = ee.FeatureCollection(miArea);
} else {
  throw new Error('MODO debe ser "wdpa", "asset" o "dibujo".');
}
var area = miAreaFC.geometry();

miAreaFC.size().evaluate(function(n) {
  if (n === 0) { print('ERROR: no se encontró el área. Revise SITE_ID (script A0) o RUTA_ASSET.'); }
});
print('Mi área – nombre(s):', miAreaFC.aggregate_array('NAME'));
print('Mi área – superficie (ha):', ee.Number(area.area(100)).divide(10000).round());

Map.centerObject(area, 8);
Map.addLayer(ee.Image().paint(miAreaFC, 0, 3), {palette: ['ffff00']}, 'Mi área');
```

**Parámetros:**
- **`typeof miArea === 'undefined'`** — verifica que exista la geometría dibujada antes de usarla; si no existe, detiene el script con un mensaje claro.
- **`area.area(100)`** — superficie con un error máximo de 100 m; dividida entre 10 000 pasa de m² a hectáreas.
- Este bloque de selección del área es **idéntico en A1, A2 y A3**.

## Paso 3 — País

```javascript
var gaul0 = ee.FeatureCollection('FAO/GAUL/2015/level0');
var paisFC = (PAIS_GAUL === null)
  ? gaul0.filterBounds(area.centroid(100)).limit(1)
  : gaul0.filter(ee.Filter.eq('ADM0_NAME', PAIS_GAUL));
var pais = paisFC.geometry();
print('País detectado (verificar):', paisFC.aggregate_array('ADM0_NAME'));
```

**Parámetros:**
- **`area.centroid(100)`** — el país se toma de GAUL según la ubicación del centro del área; funciona igual en los tres modos.
- **`limit(1)`** — si el centroide cayera en una zona disputada o en un solape de países, se queda con uno solo.

## Paso 4 — Áreas protegidas del país

```javascript
var apPais = wdpa
  .filterBounds(pais)
  .filter(ee.Filter.inList('STATUS', ['Proposed', 'Not Reported']).not())
  .filter(ee.Filter.neq('DESIG_ENG', 'UNESCO-MAB Biosphere Reserve'));

var enPais = ee.Image(0).paint(paisFC, 1);
var esAP = ee.Image(0).paint(apPais, 1);
var esMiArea = ee.Image(0).paint(miAreaFC, 1);
var haPixel = ee.Image.pixelArea().divide(10000);

var ecorregiones = ee.FeatureCollection('RESOLVE/ECOREGIONS/2017');
```

**Parámetros:**
- **Criterios de Protected Planet:** se excluyen las áreas propuestas, las de estado no reportado y las Reservas de Biosfera UNESCO-MAB, porque incluyen zonas de amortiguamiento y transición que en muchos casos no son áreas protegidas.
- **Áreas solo como punto:** no se incluyen (Protected Planet las convierte en círculos según su superficie reportada), por lo que la cobertura puede quedar levemente subestimada.
- **Máscaras ráster (`paint`):** cada máscara vale 1 donde aplica y 0 donde no. Como se trabaja con raster, las superposiciones entre áreas se cuentan **una sola vez**.
- **`haPixel`** — área de cada píxel en hectáreas; al multiplicarla por una máscara se obtiene la superficie correspondiente.

## Paso 5 — Función común: tabla por ecorregión

```javascript
function clave(id) { return ee.Number(id).int().format(); }

function tablaPorEcorregion(ecoFC, region) {
  var ecoId = ecoFC.reduceToImage({properties: ['ECO_ID'], reducer: ee.Reducer.first()}).toInt().rename('eco_id');
  var pila = haPixel.rename('ha_eco_pais')
    .addBands(haPixel.multiply(esAP).rename('ha_protegida'))
    .addBands(haPixel.multiply(esMiArea).rename('ha_mi_area'))
    .addBands(ecoId)
    .updateMask(enPais);
  var r = pila.reduceRegion({
    reducer: ee.Reducer.sum().repeat(3).group({groupField: 3, groupName: 'eco_id'}),
    geometry: region,
    scale: ESCALA,
    maxPixels: 1e11,
    tileScale: 16
  });
  var nombres = ee.Dictionary.fromLists(ecoFC.aggregate_array('ECO_ID').map(clave), ecoFC.aggregate_array('ECO_NAME'));
  var biomas = ee.Dictionary.fromLists(ecoFC.aggregate_array('ECO_ID').map(clave), ecoFC.aggregate_array('BIOME_NAME'));
  return ee.FeatureCollection(ee.List(r.get('groups')).map(function(g) {
    g = ee.Dictionary(g);
    var suma = ee.List(g.get('sum'));
    var haEco = ee.Number(suma.get(0));
    var haProt = ee.Number(suma.get(1));
    var haMia = ee.Number(suma.get(2));
    var k = clave(g.get('eco_id'));
    return ee.Feature(null, {
      ecorregion: nombres.get(k),
      bioma: biomas.get(k),
      ha_ecorregion_en_pais: haEco.round(),
      ha_protegida: haProt.round(),
      ha_no_protegida: haEco.subtract(haProt).round(),
      pct_protegido: haProt.divide(haEco).multiply(100),
      ha_en_mi_area: haMia.round(),
      pct_ecorregion_en_mi_area: haMia.divide(haEco).multiply(100)
    });
  })).sort('ha_ecorregion_en_pais', false);
}
```

**Parámetros:**
- **`reduceToImage({properties: ['ECO_ID'], ...})`** — convierte las ecorregiones vectoriales en una imagen donde cada píxel lleva el `ECO_ID` de su ecorregión.
- **Orden de bandas:** las bandas de datos van primero (`ha_eco_pais`, `ha_protegida`, `ha_mi_area`) y la banda de agrupación al final (`eco_id`). Es un requisito de `ee.Reducer.group`: `groupField: 3` apunta al índice de la última banda.
- **`ee.Reducer.sum().repeat(3)`** — suma las tres bandas de datos a la vez, por grupo.
- **`updateMask(enPais)`** — limita el cálculo a los píxeles dentro del país, de modo que `ha_ecorregion_en_pais` mide solo la parte de cada ecorregión que cae en el país.
- **`clave(id)`** — convierte el `ECO_ID` numérico en texto para usarlo como llave en los diccionarios de nombres y biomas.
- **`pct_protegido`** = hectáreas protegidas ÷ hectáreas de la ecorregión en el país × 100; **`pct_ecorregion_en_mi_area`** = hectáreas dentro de mi área ÷ hectáreas de la ecorregión en el país × 100.

```javascript
function imprimirEcorregiones(tabla, titulo, incluirMiArea) {
  var filas = tabla.reduceColumns(ee.Reducer.toList(6),
    ['ecorregion', 'ha_ecorregion_en_pais', 'ha_protegida', 'pct_protegido', 'ha_en_mi_area', 'pct_ecorregion_en_mi_area'])
    .get('list');
  ee.List(filas).evaluate(function(lista) {
    print(titulo);
    if (!lista || lista.length === 0) { print('   (sin resultados)'); return; }
    for (var i = 0; i < lista.length; i++) {
      var r = lista[i];
      var linea = '   ' + r[0] + ': extensión en el país = ' + r[1].toLocaleString() + ' ha | protegida = ' +
        r[2].toLocaleString() + ' ha (' + r[3].toFixed(1) + ' %)';
      if (incluirMiArea) {
        linea += ' | dentro de mi área = ' + r[4].toLocaleString() + ' ha (' + r[5].toFixed(2) + ' % de la ecorregión)';
      }
      print(linea);
    }
  });
}
```

**Parámetros:**
- Imprime una línea por ecorregión con todos los valores, para que cada participante pueda hacer sus propios cálculos.
- **`incluirMiArea`** — activa las columnas de "dentro de mi área" (solo tiene sentido en el escenario A).

## Paso 6 — Escenario A: ecorregiones de mi área

```javascript
var ecoMiArea = ecorregiones.filterBounds(area);
var regionA = ecoMiArea.geometry().bounds().intersection(pais.bounds(), 1000);
var tablaA = tablaPorEcorregion(ecoMiArea, regionA);

imprimirEcorregiones(tablaA, 'ESCENARIO A – Ecorregiones de mi área (valores en el país):', true);
print('Escenario A – tabla completa', tablaA);

print(ui.Chart.feature.byFeature(tablaA, 'ecorregion', ['pct_protegido', 'pct_ecorregion_en_mi_area'])
  .setChartType('BarChart')
  .setOptions({
    title: 'Ecorregiones de mi área: % protegido en el país y % que cae en mi área',
    hAxis: {title: '%', minValue: 0, maxValue: 100},
    series: {0: {labelInLegend: '% protegido en el país'}, 1: {labelInLegend: '% dentro de mi área'}}
  }));
```

**Parámetros:**
- **`ecorregiones.filterBounds(area)`** — solo las ecorregiones que intersectan el área elegida.
- **`regionA`** — rectángulo (`bounds()`) en lugar del polígono real: el cálculo es más rápido y, como `enPais` ya enmascara lo que queda fuera del país, el resultado no cambia.
- **Lectura:** la tabla responde, para cada ecorregión de mi área, cuánto del país ocupa, cuánto de eso está protegido y qué parte aporta mi área.

## Paso 7 — Escenario B: todo el país

```javascript
var tablaB = null;
if (CALCULAR_PAIS) {
  var totalPais = haPixel.rename('ha_pais').addBands(haPixel.multiply(esAP).rename('ha_protegida'))
    .updateMask(enPais)
    .reduceRegion({reducer: ee.Reducer.sum(), geometry: pais.bounds(), scale: ESCALA, maxPixels: 1e11, tileScale: 16});
  var haPais = ee.Number(totalPais.get('ha_pais'));
  var haProtPais = ee.Number(totalPais.get('ha_protegida'));
  ee.Dictionary({pais: haPais.round(), prot: haProtPais.round(), pct: haProtPais.divide(haPais).multiply(100)})
    .evaluate(function(d) {
      print('ESCENARIO B – Total del país: superficie = ' + d.pais.toLocaleString() + ' ha | protegida (WDPA) = ' +
        d.prot.toLocaleString() + ' ha (' + d.pct.toFixed(1) + ' %)');
    });

  var ecoPais = ecorregiones.filterBounds(pais);
  tablaB = tablaPorEcorregion(ecoPais, pais.bounds());
  imprimirEcorregiones(tablaB, 'ESCENARIO B – Todas las ecorregiones del país:', false);
  print('Escenario B – tabla completa', tablaB);

  print(ui.Chart.feature.byFeature(tablaB, 'ecorregion', ['pct_protegido'])
    .setChartType('BarChart')
    .setOptions({
      title: '% protegido de cada ecorregión del país (referencia Meta 3: 30 %)',
      hAxis: {title: '% protegido', minValue: 0, maxValue: 100},
      legend: {position: 'none'}
    }));
} else {
  print('Escenario B (país) desactivado. Para calcularlo, poner CALCULAR_PAIS = true.');
}
```

**Parámetros:**
- **Total nacional:** se calcula con la misma escala y la misma máscara que las ecorregiones, para que los números cuadren entre sí.
- **`pais.bounds()`** — igual que en el escenario A, el rectángulo acelera el cálculo; la máscara `enPais` hace el recorte real.
- **Lectura:** la Meta 3 busca 30 % de áreas protegidas y conservadas, ecológicamente representativas. Una ecorregión por debajo de 30 % en el país está subrepresentada; si mi área está en ella, su aporte pesa más.
- **Advertencia:** la WDPA puede no coincidir con el registro nacional; contrastar con la cifra oficial antes de reportar.

## Paso 8 — Mapa

```javascript
Map.addLayer(ecoMiArea.style({color: '666666', fillColor: '00000000', width: 1}), {}, 'Ecorregiones de mi área');
Map.addLayer(esAP.selfMask().updateMask(enPais), {palette: ['1a9850']}, 'Áreas protegidas del país (WDPA)', false);
```

**Parámetros:**
- **`fillColor: '00000000'`** — relleno totalmente transparente: solo se ve el contorno gris de las ecorregiones.
- **`selfMask()`** — enmascara los ceros para que solo se pinte lo protegido (la capa queda apagada por defecto).

## Paso 9 — Exportar tablas (opcional)

```javascript
// Export.table.toDrive({collection: tablaA, description: 'representatividad_mi_area', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
// Export.table.toDrive({collection: tablaPorEcorregion(ecorregiones.filterBounds(pais), pais.bounds()),
//   description: 'representatividad_pais', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
```

**Parámetros:**
- Si el escenario B se demora en la consola, la exportación corre en segundo plano (pestaña **Tasks**) con más capacidad. Ambos bloques están comentados por defecto.

## Notas metodológicas generales

- **Un solo reductor agrupado:** en lugar de recorrer cada ecorregión con `map` y un `reduceRegion` por ecorregión, el cálculo se hace con una sola llamada agrupada. Es lo que permite que el escenario B corra a escala de país.
- **Superposiciones:** al trabajar con máscaras ráster, dos áreas protegidas que se superponen cuentan la hectárea una sola vez; con operaciones vectoriales habría que disolver las geometrías primero.
- **Alcance del indicador:** el script calcula cobertura y representatividad por ecorregión; no evalúa efectividad de manejo ni conectividad (esta última se aborda en A3).
- **Áreas marinas:** la superficie protegida se calcula solo dentro del polígono GAUL del país, así que la parte marina de las áreas protegidas queda fuera del cálculo.

## Script "`A1_Representatividad_Ecorregiones`":
https://code.earthengine.google.com/?scriptPath=users%2Fpaulapaz1101%2Fbiodiversity_and_ecosystem_workshop%3ALima_Nov_2026%2FA1_Representatividad_Ecorregiones
