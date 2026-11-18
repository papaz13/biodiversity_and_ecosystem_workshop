---
layout: page
title: A3_Aislamiento_Conectividad
parent: "01-GoogleEarthEngine"
nav_order: 5
---

## A3_Aislamiento_Conectividad

> **Nota:** el front matter (`parent`, `nav_order`) sigue el mismo supuesto de las páginas anteriores (sección "Taller Lima 2026"); ajustalo según tu índice.

Este script describe el **aislamiento y el paisaje** de un área protegida en tres niveles. El **nivel A1** lista las áreas protegidas vecinas y resume el paisaje dentro del área y en una franja alrededor. El **nivel A2** (opcional) calcula un **corredor de menor costo** hacia el área protegida elegida. El **escenario B** resume el paisaje a nivel país, dentro y fuera de áreas protegidas, y exporta una tabla con el aislamiento de cada área protegida del país.

## Objetivos

1. Identificar las áreas protegidas vecinas y su distancia al área elegida.
2. Caracterizar el paisaje dentro del área y en su franja: porcentaje de tierra natural, presión humana 2020 y cambio de presión 2000–2020.
3. (Opcional) Calcular el corredor de menor costo hacia un área protegida de destino.
4. A nivel país, comparar el paisaje dentro y fuera de áreas protegidas.
5. Exportar una tabla por área protegida con distancia a la vecina más cercana, colindantes y presión en su franja.

**Aporta a (Marco Mundial de Biodiversidad Kunming-Montreal):** Meta A (conectividad), Meta 3 ("bien conectadas") e insumo para planificación (Indicador principal 1.1). **No calcula el indicador ProtConn.**

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

## Métodos

1. Búsqueda de áreas vecinas con `filterBounds` sobre un buffer, con cálculo de distancia y solape por entidad.
2. Resumen del paisaje (tierra natural, presión humana, cambio de presión) con `reduceRegion` y `ee.Reducer.mean()`.
3. Superficie de resistencia: combinación ponderada de presión humana y tierra no natural, escalada de 1 a 1000.
4. Costo acumulado con `cumulativeCost` desde cada área (origen y destino); la suma de ambos costos delimita el corredor.
5. Resumen nacional con promedios agrupados por zona (dentro/fuera de áreas protegidas).
6. Tabla por área protegida (`map` con reducciones anidadas), exportada a CSV.

## Paso 1 — Parámetros

```javascript
var MODO = 'wdpa';
var SITE_ID = 3370;
var RUTA_ASSET = 'projects/SU-PROYECTO/assets/mi_area';
var FRANJA_KM = 10;
var BUSQUEDA_KM = 100;

var SITE_ID_DESTINO = null;
var DIST_MIN_DESTINO_KM = 1;
var CALCULAR_CORREDOR = false;

var PAIS_GAUL = null;
var CALCULAR_PAIS = false;
```

**Parámetros:**
- **`MODO`, `SITE_ID`, `RUTA_ASSET`, `PAIS_GAUL`** — igual que en A1 y A2.
- **`BUSQUEDA_KM = 100`** — radio para buscar áreas protegidas vecinas.
- **`SITE_ID_DESTINO`** — destino del corredor (nivel A2). Opciones, en este orden de prioridad: (a) un número de `SITE_ID`, elegido de la lista impresa en el paso 3; (b) un polígono dibujado y llamado `destinoDibujado`; (c) `null` y sin dibujo: se usa la más cercana que esté a más de `DIST_MIN_DESTINO_KM`.
- **`DIST_MIN_DESTINO_KM = 1`** — por debajo de esta distancia un área se considera **colindante** (ya conectada físicamente) y se descarta como destino automático.
- **`CALCULAR_CORREDOR`** — el cálculo del corredor es más lento; lo corre primero el instructor.
- **`CALCULAR_PAIS`** — activa el resumen nacional y crea la exportación por área protegida.

## Paso 2 — Mi área, franja y capas de paisaje

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

print('Mi área – nombre(s):', miAreaFC.aggregate_array('NAME'));
print('Mi área – superficie (ha):', ee.Number(area.area(100)).divide(10000).round());

Map.centerObject(area, 8);
Map.addLayer(ee.Image().paint(miAreaFC, 0, 3), {palette: ['ffff00']}, 'Mi área');
Map.addLayer(ee.Image().paint(ee.FeatureCollection([ee.Feature(franja)]), 0, 1), {palette: ['ffffff']}, 'Franja de ' + FRANJA_KM + ' km', false);

var natural = ee.Image('WRI/SBTN/naturalLands/v1_1/2020').select('natural');
var hmSerie = ee.ImageCollection('projects/sat-io/open-datasets/GHM/HM_1990_2020_OVERALL_300M');
var hm2000 = hmSerie.filter(ee.Filter.eq('year', 2000)).first().select(0);
var hm2020 = hmSerie.filter(ee.Filter.eq('year', 2020)).first().select(0);
var pilaPaisaje = natural.multiply(100).rename('pct_natural_2020')
  .addBands(hm2020.rename('presion_humana_2020'))
  .addBands(hm2020.subtract(hm2000).rename('cambio_presion_2000_2020'));

function fx(v, d) { return (v === null || v === undefined) ? 's/d' : v.toFixed(d); }
function signo(v) { return (v === null || v === undefined) ? 's/d' : (v >= 0 ? '+' : '') + v.toFixed(3); }

var filtroEstado = ee.Filter.inList('STATUS', ['Proposed', 'Not Reported']).not();
```

**Parámetros:**
- **`pilaPaisaje`** — tres bandas que se resumen juntas en todos los niveles: porcentaje de tierra natural (el binario × 100), presión humana 2020 y cambio de presión entre 2000 y 2020 (positivo = más presión).
- **`fx` y `signo`** — formatean números para la consola; devuelven `s/d` (sin dato) cuando el valor es nulo, por ejemplo en áreas marinas.
- **`filtroEstado`** — criterio común: sin áreas propuestas ni de estado no reportado.

## Paso 3 — Nivel A1: áreas vecinas

```javascript
var vecinas = wdpa.filterBounds(area.buffer(BUSQUEDA_KM * 1000, 1000)).filter(filtroEstado);
if (MODO === 'wdpa') {
  vecinas = vecinas.filter(ee.Filter.neq('SITE_ID', SITE_ID));
}
vecinas = vecinas.map(function(f) {
  var solape = f.geometry().intersection(area, 100).area(100).divide(f.geometry().area(100));
  var distKm = f.geometry().distance(area, 100).divide(1000);
  return f.set('solape_con_mi_area', solape).set('distancia_km', distKm);
}).filter(ee.Filter.lt('solape_con_mi_area', 0.5)).sort('distancia_km');

var filasVecinas = vecinas.limit(10).map(function(f) {
  return f.set({_n: ee.Algorithms.If(f.get('NAME'), f.get('NAME'), '-'), _d: ee.Algorithms.If(f.get('DESIG'), f.get('DESIG'), '-')});
}).reduceColumns(ee.Reducer.toList(4), ['SITE_ID', '_n', '_d', 'distancia_km']).get('list');
ee.List(filasVecinas).evaluate(function(lista) {
  print('ÁREAS PROTEGIDAS VECINAS (las 10 más cercanas; para elegir destino, copiar su SITE_ID en SITE_ID_DESTINO):');
  if (!lista || lista.length === 0) {
    print('   No hay áreas protegidas de la WDPA a menos de ' + BUSQUEDA_KM + ' km. Aumente BUSQUEDA_KM.');
    return;
  }
  for (var i = 0; i < lista.length; i++) {
    var r = lista[i];
    var estado = (r[3] < DIST_MIN_DESTINO_KM) ? '  ← colindante (ya conectada)' : '';
    print('   ' + (i + 1) + '.  SITE_ID ' + r[0] + '  –  ' + r[1] + '  (' + r[2] + ')  a ' + fx(r[3], 1) + ' km' + estado);
  }
});
Map.addLayer(vecinas.limit(10).style({color: '00ffff', fillColor: '00ffff22', width: 2}), {}, 'Áreas protegidas vecinas');
```

**Parámetros:**
- **Exclusión de mi propia área:** en modo `wdpa` por su `SITE_ID`; en todos los modos, cualquier polígono que comparta más de la mitad de su superficie con mi área (por ejemplo, el mismo parque cargado como asset).
- **`distancia_km`** — distancia entre geometrías; 0 indica que se tocan. Las áreas a menos de `DIST_MIN_DESTINO_KM` se marcan como colindantes.
- **`sort('distancia_km')`** — ordena de la más cercana a la más lejana; la lista imprime las 10 primeras.

> **Nota técnica — criterio inconsistente con A1 y con el escenario B:** en este nivel `vecinas` solo aplica `filtroEstado`; **no** excluye las Reservas de Biosfera UNESCO-MAB. El escenario B de este mismo script, A1 y A2 sí las excluyen. Como consecuencia, una reserva de biosfera puede aparecer como vecina y ser elegida automáticamente como destino del corredor, aunque en el resto del análisis no cuente como área protegida. Para unificar el criterio, hay que agregar `.filter(ee.Filter.neq('DESIG_ENG', 'UNESCO-MAB Biosphere Reserve'))` a la definición de `vecinas`.

## Paso 4 — Nivel A1: paisaje dentro del área y en la franja

```javascript
function resumir(geom, nombre) {
  var r = pilaPaisaje.reduceRegion({reducer: ee.Reducer.mean(), geometry: geom, scale: 100, maxPixels: 1e10, tileScale: 8});
  return ee.Feature(null, r).set('zona', nombre);
}
var tablaPaisaje = ee.FeatureCollection([
  resumir(area, 'Dentro de mi área'),
  resumir(franja, 'Franja de ' + FRANJA_KM + ' km')
]);
tablaPaisaje.reduceColumns(ee.Reducer.toList(4), ['zona', 'pct_natural_2020', 'presion_humana_2020', 'cambio_presion_2000_2020'])
  .get('list').evaluate(function(lista) {
    print('PAISAJE (promedios):');
    for (var i = 0; i < lista.length; i++) {
      var r = lista[i];
      print('   ' + r[0] + ': tierra natural = ' + fx(r[1], 1) + ' % | presión humana 2020 = ' + fx(r[2], 3) +
        ' | cambio de presión 2000–2020 = ' + signo(r[3]));
    }
  });

var paletaHM = ['4c6100', 'adda25', 'e2ff9b', 'ffff73', 'ffe629', 'ffd37f', 'ffaa00', 'e69808', 'e60000', 'a80000', '730000'];
var vista = area.buffer(BUSQUEDA_KM * 1000, 1000);
Map.addLayer(natural.clip(vista), {min: 0, max: 1, palette: ['d7191c', '1a9641']}, 'Natural (verde) / no natural (rojo) – SBTN 2020', false);
Map.addLayer(hm2020.clip(vista), {min: 0, max: 1, palette: paletaHM}, 'Presión humana 2020 (gHM)', false);
Map.addLayer(hm2020.subtract(hm2000).clip(vista), {min: -0.3, max: 0.3, palette: ['2166ac', 'f7f7f7', 'b2182b']}, 'Cambio de presión 2000–2020', false);
```

**Parámetros:**
- **`scale: 100`** — resolución de resumen; el gHM es de 300 m y la capa de tierras naturales de 30 m, así que 100 m es un compromiso razonable para promedios por área.
- **Lectura:** una franja con poca tierra natural y presión en aumento indica que el área se está quedando aislada. Más presión dentro que en la franja es una señal de alerta sobre el estado del área.
- **`paletaHM`** — paleta de los autores del gHM (de verde oscuro a rojo oscuro).

## Paso 5 — Nivel A2 (opcional): destino del corredor

```javascript
var destinoFC;
if (SITE_ID_DESTINO !== null) {
  destinoFC = wdpa.filter(ee.Filter.eq('SITE_ID', SITE_ID_DESTINO));
} else if (typeof destinoDibujado !== 'undefined') {
  destinoFC = ee.FeatureCollection(destinoDibujado).map(function(f) { return f.set('NAME', 'Destino dibujado'); });
} else {
  destinoFC = ee.FeatureCollection(vecinas.filter(ee.Filter.gt('distancia_km', DIST_MIN_DESTINO_KM)).limit(1));
}
print('Destino del corredor:', destinoFC.aggregate_array('NAME'),
  'Distancia a mi área (km):', ee.Number(destinoFC.geometry().distance(area, 100)).divide(1000));
Map.addLayer(ee.Image().paint(destinoFC, 0, 3), {palette: ['ff00ff']}, 'Destino del corredor');
```

**Parámetros:**
- **Prioridad de elección:** `SITE_ID_DESTINO` numérico, luego polígono `destinoDibujado`, luego la vecina más cercana no colindante.
- **Descartar colindantes:** un área que toca a la mía ya está conectada físicamente; un corredor hacia ella no aporta información.

## Paso 6 — Nivel A2 (opcional): corredor de menor costo

```javascript
if (CALCULAR_CORREDOR) {
  var ESCALA = 250;
  var proyeccion = ee.Projection('EPSG:4326').atScale(ESCALA);
  var region = area.union(destinoFC.geometry(), 1000).buffer(10000, 1000).bounds(1000);

  var nodos = miAreaFC.merge(destinoFC);
  var resistencia = hm2020.unmask(0).multiply(0.6)
    .add(ee.Image(1).subtract(natural).unmask(1).multiply(0.4))
    .multiply(999).add(1)
    .where(ee.Image(0).paint(nodos, 1).eq(1), 1)
    .clip(region)
    .reproject(proyeccion);

  var DIST_MAX = (2 * BUSQUEDA_KM + 50) * 1000;
  var costoDesde = function(fc) {
    var fuente = ee.Image().toByte().paint(fc, 1).selfMask();
    return resistencia.cumulativeCost({source: fuente, maxDistance: DIST_MAX}).reproject(proyeccion);
  };
  var costoTotal = costoDesde(miAreaFC).add(costoDesde(destinoFC)).rename('costo');

  var minimo = ee.Number(costoTotal.reduceRegion({
    reducer: ee.Reducer.min(), geometry: region, crs: proyeccion, maxPixels: 1e9
  }).get('costo'));

  var corredor = costoTotal.lte(minimo.multiply(1.05)).selfMask();
  var ruta = costoTotal.lte(minimo.multiply(1.002)).selfMask();

  Map.addLayer(resistencia, {min: 1, max: 1000, palette: ['1a9850', 'fee08b', 'd73027']}, 'Resistencia', false);
  Map.addLayer(corredor, {palette: ['ff00ff']}, 'Corredor (≤ 5 % sobre el mínimo)');
  Map.addLayer(ruta, {palette: ['000000']}, 'Ruta aproximada de menor costo');

  var resumenCorredor = ee.Image.pixelArea().divide(10000).rename('ha')
    .addBands(hm2020.rename('presion'))
    .addBands(natural.rename('natural'))
    .updateMask(corredor)
    .reduceRegion({reducer: ee.Reducer.sum().combine(ee.Reducer.mean(), null, true), geometry: region, crs: proyeccion, maxPixels: 1e9});
  ee.Dictionary(resumenCorredor).evaluate(function(d) {
    print('CORREDOR: superficie = ' + Math.round(d.ha_sum).toLocaleString() + ' ha | presión humana media = ' +
      fx(d.presion_mean, 3) + ' | tierra natural = ' + fx(d.natural_mean === null ? null : d.natural_mean * 100, 1) + ' %');
  });
} else {
  print('Corredor (A2) desactivado. Para calcularlo, poner CALCULAR_CORREDOR = true y volver a correr.');
}
```

**Parámetros:**
- **Superficie de resistencia:** 60 % presión humana + 40 % tierra no natural, escalada de 1 a 1000. Dentro de los dos nodos (origen y destino) la resistencia es mínima (1). Los pesos son fijos y no se ajustan en el taller; es una superficie **genérica, no calibrada para una especie**.
- **`ESCALA = 250`** — el costo acumulado es pesado; se trabaja a 250 m en una proyección común (`proyeccion`).
- **`costoDesde(fc)`** — costo del camino más barato desde cada píxel hasta la fuente más cercana (`ee.Image.cumulativeCost`, guía oficial "Cumulative Cost Mapping").
- **`costoTotal`** — suma del costo desde mi área y del costo desde el destino. El valor mínimo de esta suma es el costo del mejor camino entre ambas áreas.
- **`corredor` y `ruta`** — píxeles cuyo camino cuesta hasta 5 % más que el mejor camino (corredor) o hasta 0.2 % más (ruta aproximada de menor costo). El 5 % es un umbral fijo de visualización, no un resultado calibrado.
- **`sum().combine(mean(), null, true)`** — con entradas compartidas, devuelve en un solo paso la suma y la media de cada banda (`ha_sum`, `presion_mean`, `natural_mean`).
- **Pregunta para discutir:** ¿en qué tramos el corredor cruza zonas de alta presión o tierra no natural?

> **Nota técnica — valores a validar en la primera corrida:** (1) `maxDistance` de `cumulativeCost` limita el cálculo con `DIST_MAX` = 250 km; con resistencias de 1 a 1000 el costo acumulado crece mucho más rápido que la distancia, por lo que conviene verificar en la guía de Earth Engine en qué unidades se interpreta ese límite y, si fuera costo acumulado, que 250 000 no trunque el cálculo antes de unir ambas áreas. (2) Los datos faltantes se tratan de forma asimétrica: `hm2020.unmask(0)` supone presión nula y `unmask(1)` en la capa natural supone tierra no natural; fuera de la cobertura de los datos (por ejemplo sobre el mar) la resistencia queda en un valor intermedio-alto (~400).

## Paso 7 — Escenario B: resumen nacional (consola, 1 km)

```javascript
var gaul0 = ee.FeatureCollection('FAO/GAUL/2015/level0');
var paisFC = (PAIS_GAUL === null)
  ? gaul0.filterBounds(area.centroid(100)).limit(1)
  : gaul0.filter(ee.Filter.eq('ADM0_NAME', PAIS_GAUL));

if (CALCULAR_PAIS) {
  print('País detectado (verificar):', paisFC.aggregate_array('ADM0_NAME'));
  var pais = paisFC.geometry();
  var apPais = wdpa.filterBounds(pais).filter(filtroEstado)
    .filter(ee.Filter.neq('DESIG_ENG', 'UNESCO-MAB Biosphere Reserve'));

  var enPais = ee.Image(0).paint(paisFC, 1);
  var dentroAP = ee.Image(0).paint(apPais, 1);
  var zonaPais = dentroAP.add(1).rename('zona').updateMask(enPais);
  var rPais = pilaPaisaje.addBands(zonaPais).reduceRegion({
    reducer: ee.Reducer.mean().repeat(3).group({groupField: 3, groupName: 'zona'}),
    geometry: pais.bounds(),
    scale: 1000,
    maxPixels: 1e10,
    tileScale: 16
  });
  ee.List(rPais.get('groups')).evaluate(function(grupos) {
    print('ESCENARIO B – Paisaje del país (promedios a 1 km):');
    for (var i = 0; i < grupos.length; i++) {
      var g = grupos[i];
      var nombre = (g.zona === 2) ? 'Dentro de áreas protegidas' : 'Fuera de áreas protegidas';
      print('   ' + nombre + ': tierra natural = ' + fx(g.mean[0], 1) + ' % | presión humana 2020 = ' +
        fx(g.mean[1], 3) + ' | cambio de presión 2000–2020 = ' + signo(g.mean[2]));
    }
  });
```

**Parámetros:**
- **`scale: 1000`** — escala suficiente para un promedio nacional que mantiene el cálculo en la consola.
- **`zonaPais`** — 1 = fuera de áreas protegidas, 2 = dentro; es la banda de agrupación (`groupField: 3`, al final de las cuatro bandas).
- **Criterio de áreas protegidas:** mismos criterios que A1 (incluye la exclusión de Reservas UNESCO-MAB).
- **Lectura:** comparar la tierra natural y la presión humana dentro y fuera de áreas protegidas muestra si la red protege lugares menos presionados o si la presión alcanza también a las áreas protegidas.

## Paso 8 — Escenario B: tabla por área protegida (exportación)

```javascript
  var tablaAP = apPais.map(function(f) {
    var g = f.geometry();
    var anillo = g.buffer(FRANJA_KM * 1000, 100).difference(g, 100);
    var otras = apPais.filterBounds(g.buffer(BUSQUEDA_KM * 1000, 1000)).filter(ee.Filter.neq('SITE_ID', f.get('SITE_ID')))
      .map(function(o) { return o.set('d', o.geometry().distance(g, 100).divide(1000)); });
    var dentro = pilaPaisaje.reduceRegion({reducer: ee.Reducer.mean(), geometry: g, scale: 300, maxPixels: 1e10, tileScale: 4});
    var fuera = pilaPaisaje.reduceRegion({reducer: ee.Reducer.mean(), geometry: anillo, scale: 300, maxPixels: 1e10, tileScale: 4});
    return ee.Feature(null, {
      SITE_ID: f.get('SITE_ID'),
      NAME: f.get('NAME'),
      DESIG: f.get('DESIG'),
      ha: ee.Number(g.area(100)).divide(10000).round(),
      n_colindantes: otras.filter(ee.Filter.lt('d', DIST_MIN_DESTINO_KM)).size(),
      dist_km_vecina_no_colindante: otras.filter(ee.Filter.gte('d', DIST_MIN_DESTINO_KM)).aggregate_min('d'),
      presion_2020_dentro: dentro.get('presion_humana_2020'),
      presion_2020_franja: fuera.get('presion_humana_2020'),
      pct_natural_franja: fuera.get('pct_natural_2020'),
      cambio_presion_franja: fuera.get('cambio_presion_2000_2020')
    });
  });
  Export.table.toDrive({
    collection: tablaAP,
    description: 'conectividad_areas_protegidas_pais',
    folder: 'GEE_taller_lima',
    fileFormat: 'CSV'
  });
  print('ESCENARIO B: tarea "conectividad_areas_protegidas_pais" creada. Ir a la pestaña Tasks y presionar RUN.');
} else {
  print('Escenario B (país) desactivado. Para calcularlo, poner CALCULAR_PAIS = true.');
}
```

**Parámetros:**
- **Tabla por área:** para cada área protegida calcula distancia a la vecina más cercana no colindante, número de colindantes, presión humana dentro, y presión y tierra natural en su franja. Es pesada para la consola, por eso se exporta (pestaña **Tasks** → **RUN**).
- **`dist_km_vecina_no_colindante`** — es el indicador de **aislamiento**: distancia a la vecina más cercana que no toca el área.
- **`scale: 300`** — resolución de las reducciones por área (el gHM es de 300 m).
- **Uso del CSV:** ordenar las áreas por aislamiento (distancia a la vecina más cercana) o por presión en su franja.

> **Nota técnica — áreas no terrestres:** `apPais` no filtra por `REALM`, así que la tabla incluye áreas marinas y costeras, para las que el paisaje terrestre tiene poco o ningún dato (resultados nulos o promedios de una franja parcial). Para una tabla estrictamente terrestre, agregar `.filter(ee.Filter.eq('REALM', 'Terrestrial'))` a `apPais`.

## Paso 9 — Exportar el escenario A (opcional)

```javascript
// Export.table.toDrive({collection: tablaPaisaje, description: 'paisaje_mi_area', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
// Export.table.toDrive({collection: vecinas.select(['SITE_ID', 'NAME', 'DESIG', 'distancia_km']).limit(10),
//   description: 'areas_vecinas_mi_area', folder: 'GEE_taller_lima', fileFormat: 'CSV'});
```

**Parámetros:**
- Ambos bloques están comentados por defecto; hay que descomentarlos para generar las tareas en **Tasks**.

## Notas metodológicas generales

- **Qué mide y qué no:** el script describe el entorno y la distancia entre áreas protegidas; **no calcula ProtConn**, que exige además una función de dispersión y el área protegida conectada ponderada. Se usa como insumo exploratorio de planificación.
- **Corredor genérico:** la superficie de resistencia no está calibrada para ninguna especie. El corredor muestra por dónde pasa el camino de menor presión humana y menor transformación, no el movimiento real de una especie.
- **Distancia entre geometrías:** la medición de aislamiento usa distancias entre bordes de polígonos de la WDPA, que no siempre coinciden con el límite oficial nacional; contrastar antes de reportar.
- **Fuente comunitaria:** el gHM depende de un catálogo comunitario; documentar la versión usada (v3, Theobald et al. 2025) en cualquier reporte.

## Script "`A3_Aislamiento_Conectividad`":
https://code.earthengine.google.com/?scriptPath=users%2Fpaulapaz1101%2Fbiodiversity_and_ecosystem_workshop%3ALima_Nov_2026%2FA3_Aislamiento_Conectividad
