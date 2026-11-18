---
layout: page
title: A2_Perdida_Cobertura_Carbono
parent: "01-GoogleEarthEngine"
nav_order: 4
---

## A2_Perdida_Cobertura_Carbono

Este script cuantifica la **pérdida de cobertura arbórea** año por año (2001–2025) dentro de un área protegida y en una franja alrededor, estima el **carbono** asociado a esa pérdida e identifica la **causa dominante**. A nivel país, calcula como exportación cuánto se perdió cada año dentro y fuera de las áreas protegidas.

## Objetivos

1. Definir el área de interés (WDPA, asset propio o dibujo) y una franja de influencia alrededor.
2. Calcular la pérdida de cobertura arbórea por año dentro del área y en la franja, usando Hansen.
3. Estimar el carbono perdido por año (desde 2011) y el stock de carbono ~2010 de la biomasa viva.
4. Identificar la causa dominante de la pérdida con el producto WRI/Google DeepMind.
5. Calcular, a nivel país, la pérdida anual dentro y fuera de áreas protegidas mediante una exportación (escenario B).

**Aporta a (Marco Mundial de Biodiversidad Kunming-Montreal):** Indicador principal A.2 (extensión de ecosistemas naturales) y Meta 1 (pérdida de áreas de alta integridad cercana a cero); el carbono asociado a la pérdida se vincula a la Meta 8.

## Datos

- **`UMD/hansen/global_forest_change_2025_v1_13`** — Cambio global de cobertura arbórea (30 m, pérdida 2001–2025).<br>
Image: `ee.Image('UMD/hansen/global_forest_change_2025_v1_13')`

- **`NASA/ORNL/biomass_carbon_density/v1`** — Densidad de carbono en biomasa viva, aérea y subterránea (300 m, año ~2010).<br>
Collection: `ee.ImageCollection('NASA/ORNL/biomass_carbon_density/v1')`

- **`projects/landandcarbon/assets/wri_gdm_drivers_forest_loss_1km/v1_3_2001_2025`** — Causas dominantes de la pérdida de cobertura arbórea, WRI/Google DeepMind (1 km, 2001–2025).<br>
Image: `ee.Image('projects/landandcarbon/assets/wri_gdm_drivers_forest_loss_1km/v1_3_2001_2025')`

- **`WCMC/WDPA/current/polygons`** — Áreas protegidas (WDPA).<br>
Collection: `ee.FeatureCollection('WCMC/WDPA/current/polygons')`

- **`FAO/GAUL/2015/level0`** — Límites de países.<br>
Collection: `ee.FeatureCollection('FAO/GAUL/2015/level0')`

> **Nota técnica:** los identificadores de Hansen y del producto de causas incluyen la versión y el período (`2025_v1_13`, `v1_3_2001_2025`). Cuando salga una versión nueva, hay que actualizar ambos identificadores y revisar el rango de años (`lossyear` de 1 a 25 en esta versión).

## Métodos

1. Definición del área y de la franja con `buffer(...).difference(area)`.
2. Pérdida de cobertura arbórea de Hansen: `lossyear` enmascarado a píxeles con cobertura en 2000 igual o mayor a un umbral.
3. Carbono perdido por píxel: densidad (Mg/ha) × área del píxel (ha), solo para pérdidas desde 2011.
4. Suma agrupada por año con `ee.Reducer.sum().repeat(2).group()` en una sola llamada por zona.
5. Stock de carbono ~2010 a la resolución nativa del producto (300 m).
6. Suma agrupada por causa dominante, ponderada por la pérdida de Hansen.
7. Escenario país: código `zona × 100 + año` agrupado en una exportación a CSV.

## Paso 1 — Parámetros

```javascript
var MODO = 'wdpa';
var SITE_ID = 3370;
var RUTA_ASSET = 'projects/SU-PROYECTO/assets/mi_area';
var FRANJA_KM = 10;
var COBERTURA_MIN = 30;
var PAIS_GAUL = null;
var CALCULAR_PAIS = false;
```

**Parámetros:**
- **`MODO`, `SITE_ID`, `RUTA_ASSET`, `PAIS_GAUL`** — igual que en A1: el área se define en uno de tres modos y el país se detecta solo salvo que se fuerce.
- **`FRANJA_KM = 10`** — ancho de la franja alrededor del área, donde se mide la presión del entorno.
- **`COBERTURA_MIN = 30`** — porcentaje mínimo de cobertura de copa en 2000 para considerar un píxel como "árbol" (valor usado por Global Forest Watch).
- **`CALCULAR_PAIS = false`** — el escenario B está apagado por defecto; con `true` crea la tarea de exportación en la pestaña **Tasks**.

## Paso 2 — Mi área y su franja

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
var franja = area.buffer(FRANJA_KM * 1000, 100).difference(area, 100);

var haArea = ee.Number(area.area(100)).divide(10000);
print('Mi área – nombre(s):', miAreaFC.aggregate_array('NAME'));
print('Mi área – superficie (ha):', haArea.round());
haArea.evaluate(function(ha) {
  if (ha > 1000000) {
    print('AVISO: el área supera 1 millón de ha. Si el cálculo se demora o falla, use la exportación del paso 8.');
  }
});

Map.centerObject(area, 9);
Map.setOptions('HYBRID');
Map.addLayer(ee.Image().paint(miAreaFC, 0, 3), {palette: ['ffff00']}, 'Mi área');
Map.addLayer(ee.Image().paint(ee.FeatureCollection([ee.Feature(franja)]), 0, 1), {palette: ['ffffff']}, 'Franja de ' + FRANJA_KM + ' km');

var gaul0 = ee.FeatureCollection('FAO/GAUL/2015/level0');
var paisFC = (PAIS_GAUL === null)
  ? gaul0.filterBounds(area.centroid(100)).limit(1)
  : gaul0.filter(ee.Filter.eq('ADM0_NAME', PAIS_GAUL));
print('País detectado (verificar):', paisFC.aggregate_array('ADM0_NAME'));
```

**Parámetros:**
- **`buffer(...).difference(area)`** — la franja es un anillo: el buffer de `FRANJA_KM` menos el área. Así el análisis separa lo que ocurre dentro del área de lo que ocurre en su entorno.
- **Aviso de 1 millón de ha:** para áreas muy grandes el cálculo en consola puede demorarse o fallar; la salida es exportar.
- Mapa base `HYBRID` (satélite con etiquetas) para ubicar las pérdidas en el terreno.

> **Nota de transcripción:** el aviso de áreas grandes remite a "la exportación del paso 8", pero en este script el paso 8 es "Cómo leer los resultados". Las exportaciones están en el paso 7 (escenario país, que ya es una exportación) y en el paso 9 (tablas del escenario A, comentadas). Conviene corregir la referencia a "paso 9" en el script fuente.

## Paso 3 — Pérdida de cobertura arbórea (Hansen)

```javascript
var gfc = ee.Image('UMD/hansen/global_forest_change_2025_v1_13');
var arboles2000 = gfc.select('treecover2000').gte(COBERTURA_MIN);
var anioPerdida = gfc.select('lossyear').updateMask(arboles2000.and(gfc.select('lossyear').gt(0)));

Map.addLayer(arboles2000.selfMask(), {palette: ['1a9850']}, 'Cobertura arbórea 2000 (≥ ' + COBERTURA_MIN + ' %)', false);
Map.addLayer(anioPerdida, {min: 1, max: 25, palette: ['ffff00', 'ff8800', 'ff0000']}, 'Pérdida por año (amarillo 2001 → rojo 2025)');
```

**Parámetros:**
- **`lossyear`** — 0 indica sin pérdida; 1 a 25 indica pérdida en 2001–2025.
- **Terminología:** Hansen registra la eliminación completa del dosel (cambio de bosque a no bosque) por cualquier causa: tala, minería, incendio o perturbación natural. Por eso se habla de **pérdida de cobertura arbórea** y no de deforestación.
- **`arboles2000.and(lossyear > 0)`** — conserva solo la pérdida ocurrida en píxeles que eran árboles en el año 2000 (cobertura mayor o igual a `COBERTURA_MIN`).

## Paso 4 — Carbono (NASA/ORNL, año 2010)

```javascript
var carbono = ee.ImageCollection('NASA/ORNL/biomass_carbon_density/v1').first();
var carbonoTotal = carbono.select('agb').add(carbono.select('bgb')).rename('carbono');
var ANIO_INICIO_CARBONO = 2011;

Map.addLayer(carbonoTotal.clip(area.buffer(FRANJA_KM * 1000)), {min: 0, max: 150, palette: ['ffffcc', '78c679', '006837']}, 'Carbono en biomasa ~2010 (Mg/ha)', false);
```

**Parámetros:**
- **`agb + bgb`** — densidad de carbono en biomasa viva, aérea (`agb`) más subterránea (`bgb`), en Mg/ha a 300 m.
- **`ANIO_INICIO_CARBONO = 2011`** — el mapa representa ~2010, de modo que la pérdida **anterior a 2011** ya no tiene carbono en el mapa. Por eso el carbono perdido se calcula solo desde 2011, mientras que las hectáreas perdidas se cuentan desde 2001.
- **Supuesto:** dentro de cada celda de 300 m la densidad de carbono es uniforme.

## Paso 5 — Escenario A: pérdida y carbono por año

```javascript
var haPixel = ee.Image.pixelArea().divide(10000);
var carbonoPorPixel = carbonoTotal.multiply(haPixel)
  .updateMask(anioPerdida.gte(ANIO_INICIO_CARBONO - 2000)).unmask(0);

var pila = haPixel.rename('ha').addBands(carbonoPorPixel.rename('carbono_Mg')).addBands(anioPerdida.rename('anio'))
  .updateMask(anioPerdida.mask());

function sumarPorAnio(geom) {
  var r = pila.reduceRegion({
    reducer: ee.Reducer.sum().repeat(2).group({groupField: 2, groupName: 'anio'}),
    geometry: geom,
    scale: 30,
    maxPixels: 1e11,
    tileScale: 8
  });
  var grupos = ee.List(r.get('groups'));
  var claves = grupos.map(function(g) { return ee.Number(ee.Dictionary(g).get('anio')).int().format(); });
  var valores = grupos.map(function(g) { return ee.Dictionary(g).get('sum'); });
  return ee.Dictionary.fromLists(claves, valores);
}

var dentro = sumarPorAnio(area);
var enFranja = sumarPorAnio(franja);
```

**Parámetros:**
- **`carbonoPorPixel`** — Mg de carbono por píxel de 30 m perdido = densidad (Mg/ha) × hectáreas del píxel; se enmascara a pérdidas de 2011 en adelante.
- **Orden de bandas:** `ha` y `carbono_Mg` primero, `anio` al final; `groupField: 2` apunta a la banda de agrupación.
- **`scale: 30`** — resolución nativa de Hansen. A 30 m no se pierden los parches pequeños de pérdida.
- **`sumarPorAnio`** — una sola llamada agrupada por zona (dentro y franja) devuelve un diccionario año → [hectáreas, Mg C].

```javascript
var ANIOS = [];
for (var a = 1; a <= 25; a++) { ANIOS.push(a); }

var tabla = ee.FeatureCollection(ANIOS.map(function(a) {
  var k = String(a);
  var d = ee.List(dentro.get(k, [0, 0]));
  var f = ee.List(enFranja.get(k, [0, 0]));
  return ee.Feature(null, {
    anio: 2000 + a,
    ha_dentro: ee.Number(d.get(0)),
    ha_franja: ee.Number(f.get(0)),
    carbono_Mg_dentro: ee.Number(d.get(1)),
    carbono_Mg_franja: ee.Number(f.get(1))
  });
}));

print('PÉRDIDA POR AÑO (ha y Mg de carbono; carbono solo desde ' + ANIO_INICIO_CARBONO + ')', tabla);

print(ui.Chart.feature.byFeature(tabla, 'anio', ['ha_dentro', 'ha_franja'])
  .setChartType('ColumnChart')
  .setOptions({
    title: 'Pérdida de cobertura arbórea por año (ha)',
    hAxis: {title: 'Año', format: '####'},
    vAxis: {title: 'ha'},
    series: {0: {labelInLegend: 'Dentro de mi área', color: 'd7191c'}, 1: {labelInLegend: 'Franja de ' + FRANJA_KM + ' km', color: 'fdae61'}}
  }));
```

**Parámetros:**
- **`dentro.get(k, [0, 0])`** — si un año no tiene pérdida, el diccionario no tiene esa llave y se usa cero por defecto, de modo que la tabla siempre tiene los 25 años.
- **`anio: 2000 + a`** — convierte el código de `lossyear` (1 a 25) en año calendario.

```javascript
var totales = tabla.reduceColumns({
  reducer: ee.Reducer.sum().repeat(4),
  selectors: ['ha_dentro', 'ha_franja', 'carbono_Mg_dentro', 'carbono_Mg_franja']
}).get('sum');
totales = ee.List(totales);
print('Pérdida total 2001–2025 dentro de mi área (ha):', ee.Number(totales.get(0)).round());
print('Pérdida total 2001–2025 en la franja (ha):', ee.Number(totales.get(1)).round());
print('Carbono perdido ' + ANIO_INICIO_CARBONO + '–2025 dentro de mi área (Mg C):', ee.Number(totales.get(2)).round());
print('Carbono perdido ' + ANIO_INICIO_CARBONO + '–2025 en la franja (Mg C):', ee.Number(totales.get(3)).round());

var stock2010 = ee.Number(carbonoTotal.multiply(haPixel).reduceRegion({
  reducer: ee.Reducer.sum(), geometry: area, scale: 300, maxPixels: 1e10, tileScale: 4
}).get('carbono'));
print('Stock de carbono en biomasa ~2010 dentro de mi área (Mg C):', stock2010.round());
print('Stock remanente estimado (stock ~2010 − carbono perdido ' + ANIO_INICIO_CARBONO + '–2025) (Mg C):',
  stock2010.subtract(ee.Number(totales.get(2))).round());
```

**Parámetros:**
- **`stock2010`** — se calcula a 300 m, la resolución nativa del producto de carbono.
- **Stock remanente** = stock ~2010 menos carbono perdido desde 2011. Es una estimación: resta una cifra calculada a 30 m (píxeles perdidos de Hansen) de otra calculada a 300 m; sirve como orden de magnitud, no como inventario.

## Paso 6 — Escenario A: causa dominante de la pérdida

```javascript
var causas = ee.Image('projects/landandcarbon/assets/wri_gdm_drivers_forest_loss_1km/v1_3_2001_2025').select('classification');
var NOMBRES_CAUSA = ee.Dictionary({
  '1': 'Agricultura permanente', '2': 'Commodities duras (ej. minería)', '3': 'Agricultura migratoria',
  '4': 'Tala', '5': 'Incendio', '6': 'Asentamientos e infraestructura', '7': 'Otras perturbaciones naturales'
});

var pilaCausas = haPixel.rename('ha').addBands(causas.rename('causa')).updateMask(anioPerdida.mask());
var porCausa = pilaCausas.reduceRegion({
  reducer: ee.Reducer.sum().group({groupField: 1, groupName: 'causa'}),
  geometry: area.buffer(FRANJA_KM * 1000, 100),
  scale: 30,
  maxPixels: 1e11,
  tileScale: 8
});
var tablaCausas = ee.FeatureCollection(ee.List(porCausa.get('groups')).map(function(g) {
  g = ee.Dictionary(g);
  var k = ee.Number(g.get('causa')).int().format();
  return ee.Feature(null, {causa: NOMBRES_CAUSA.get(k, 'Sin clasificar'), ha_perdidas: ee.Number(g.get('sum')).round()});
}));
print('Pérdida 2001–2025 por causa dominante (área + franja, ha)', tablaCausas);
print(ui.Chart.feature.byFeature(tablaCausas, 'causa', ['ha_perdidas'])
  .setChartType('PieChart').setOptions({title: 'Causa dominante de la pérdida (área + franja)'}));

Map.addLayer(causas.clip(area.buffer(FRANJA_KM * 1000)), {min: 1, max: 7,
  palette: ['e39d29', 'e9d700', '51a44e', '895128', 'a354a0', '3a209a', '78d3ac']}, 'Causa dominante (1 km)', false);
```

**Parámetros:**
- **Naturaleza del producto:** es **una** imagen resumen 2001–2025 a 1 km: la causa dominante de la pérdida en cada celda, no una causa por año.
- **Ponderación:** cada píxel de Hansen (30 m) hereda la causa de la celda de 1 km donde cae; la suma por causa son las hectáreas de pérdida de Hansen atribuidas a cada causa dominante. Si una celda tiene varias causas, toda su pérdida se asigna a la dominante.
- **Geometría:** área más franja (`area.buffer(...)`), porque el análisis de causas describe el contexto completo de presión.
- **`NOMBRES_CAUSA.get(k, 'Sin clasificar')`** — valores fuera del diccionario se rotulan como sin clasificar.

## Paso 7 — Escenario B: todo el país, dentro y fuera de áreas protegidas

```javascript
if (CALCULAR_PAIS) {
  var apPais = wdpa
    .filterBounds(paisFC.geometry())
    .filter(ee.Filter.inList('STATUS', ['Proposed', 'Not Reported']).not())
    .filter(ee.Filter.neq('DESIG_ENG', 'UNESCO-MAB Biosphere Reserve'));
  var enPais = ee.Image(0).paint(paisFC, 1);
  var zona = ee.Image(1).where(ee.Image(0).paint(apPais, 1).eq(1), 2);
  var codigo = zona.multiply(100).add(anioPerdida).rename('codigo');
  var pilaPais = haPixel.rename('ha').addBands(carbonoPorPixel.rename('carbono_Mg')).addBands(codigo)
    .updateMask(anioPerdida.mask()).updateMask(enPais);
  var rPais = pilaPais.reduceRegion({
    reducer: ee.Reducer.sum().repeat(2).group({groupField: 2, groupName: 'codigo'}),
    geometry: paisFC.geometry().bounds(),
    scale: 30,
    maxPixels: 1e13,
    tileScale: 16
  });
  var tablaPais = ee.FeatureCollection(ee.List(rPais.get('groups')).map(function(g) {
    g = ee.Dictionary(g);
    var c = ee.Number(g.get('codigo')).int();
    var suma = ee.List(g.get('sum'));
    return ee.Feature(null, {
      anio: c.mod(100).add(2000),
      zona: ee.Algorithms.If(c.divide(100).int().eq(2), 'Dentro de áreas protegidas', 'Fuera de áreas protegidas'),
      ha_perdidas: ee.Number(suma.get(0)).round(),
      carbono_Mg: ee.Number(suma.get(1)).round()
    });
  })).sort('anio');
  Export.table.toDrive({
    collection: tablaPais,
    description: 'perdida_pais_dentro_fuera_AP',
    folder: 'GEE_taller_lima',
    fileFormat: 'CSV'
  });
  print('ESCENARIO B: tarea "perdida_pais_dentro_fuera_AP" creada. Ir a la pestaña Tasks y presionar RUN.');
} else {
  print('Escenario B (país) desactivado. Para crear la exportación, poner CALCULAR_PAIS = true.');
}
```

**Parámetros:**
- **Por qué es una exportación:** a 30 m un país entero tiene miles de millones de píxeles y no cabe en la consola (límite de ~5 minutos). La exportación corre en segundo plano con más capacidad. En países grandes puede tardar mucho; conviene que el instructor la corra antes del taller.
- **No se baja a 300 m:** la pérdida de Hansen es de 30 m y se perderían los parches pequeños.
- **Código de agrupación:** `zona × 100 + año`, con zona 1 = fuera de áreas protegidas y 2 = dentro. Un código 215 significa "dentro de áreas protegidas, pérdida en 2015". Al leer el resultado se separan con `mod(100)` (año) y `divide(100)` (zona).
- **Áreas protegidas:** mismos criterios que en A1 (sin propuestas, sin estado no reportado, sin Reservas UNESCO-MAB).
- **Uso del CSV:** permite calcular, por ejemplo, qué porcentaje de la pérdida nacional ocurrió dentro de áreas protegidas, o qué porcentaje representa mi área (escenario A) respecto del total del país.

## Paso 8 — Cómo leer los resultados

- Hansen sirve para tendencias anuales; no es un mapa de alertas ni reemplaza la verificación en campo.
- El carbono es una estimación: densidad ~2010 a 300 m aplicada a píxeles perdidos de 30 m, solo desde 2011.
- En áreas sin bosque (puna, páramo, lomas, humedales) la pérdida de cobertura arbórea será cercana a cero.
- La WDPA puede no coincidir con el registro nacional de áreas protegidas.

## Paso 9 — Exportar el escenario A (opcional)

```javascript
// Export.table.toDrive({collection: tabla, description: 'perdida_por_anio_mi_area', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
// Export.table.toDrive({collection: tablaCausas, description: 'perdida_por_causa_mi_area', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
```

**Parámetros:**
- Ambos bloques están comentados por defecto; hay que descomentarlos para generar las tareas en **Tasks**.

## Notas metodológicas generales

- **Dos escalas, una sola cifra:** las hectáreas se miden a 30 m y el carbono se asigna desde una capa de 300 m. Las tendencias anuales son robustas; el valor absoluto de carbono por píxel no lo es.
- **Años sin carbono:** el carbono perdido 2001–2010 no se estima, porque el mapa de densidad ya refleja la condición posterior a esas pérdidas. Por eso `ha_dentro` suma desde 2001 y `carbono_Mg_dentro` desde 2011.
- **Pérdida no es deforestación:** Hansen incluye pérdida por incendio y por perturbaciones naturales, además de cambios de uso del suelo. La causa dominante ayuda a interpretar, pero se resuelve a 1 km.
- **Meta 8:** el carbono asociado a una pérdida fechada (2011–2025) se vincula a la Meta 8 como acción de mitigación ligada a un evento discreto, no como flujo continuo de servicio ecosistémico.

## Script "`A2_Perdida_Cobertura_Carbono`":
- https://code.earthengine.google.com/?scriptPath=users%2Fpaulapaz1101%2Fbiodiversity_and_ecosystem_workshop%3ALima_Nov_2026%2FA2_Perdida_Cobertura_Carbono
